# Vooronderzoek: Speckle Transport & QGIS-BIM Integration

> Research document for the Cross-Tech AEC Skill Package.
> Covers Speckle architecture, connectors (Blender, Revit), SpecklePy API, and QGIS-BIM/IFC integration patterns.
> Last updated: 2026-03-20

---

## 1. Speckle Architecture (Server 2.x / 3.x)

### 1.1 Core Concepts

Speckle is a versioned object graph storage system for AEC data — not a file-based system. It decomposes 3D models into atomic constituent parts, hashes each part, and stores them across multiple persistence layers. The core architecture consists of four pillars:

**Kits** define a shared schema — a common language for AEC data exchange. Each kit includes translation routines that convert between application-native formats and Speckle's shared schema. Kits are hot-swappable, meaning users can customize the conversion layer while maintaining interoperability across tools.

**Transports** define how Speckle writes to and reads from a given persistence layer. This abstraction layer decouples the data model from storage, allowing objects to flow simultaneously to multiple destinations (server, local SQLite cache, memory, disk).

**Serialization & Decomposition** breaks objects into constituent parts during serialization. Each object receives a unique hash-based ID, making objects immutable. Objects in Speckle are essentially both a tree and a blob at the same time. JSON is the native serialization format.

**Connectors** are plugins for authoring applications (Revit, Blender, Rhino, QGIS, AutoCAD) that handle conversion between native application objects and Speckle's shared schema.

### 1.2 Data Model Hierarchy

Speckle organizes data hierarchically:

| Level | Description |
|-------|-------------|
| **Workspace** | Top-level organizational unit (v3) |
| **Project** (formerly Stream) | A container for models and versions |
| **Model** (formerly Branch) | A named channel within a project |
| **Version** (formerly Commit) | An immutable snapshot pointing to a root object |
| **Object** | A Base instance with a unique hash-based ID |

Every change creates a new version, providing full history and traceability. Objects not associated with any version become orphans and are effectively irretrievable.

### 1.3 The Base Object

The `Base` class is the foundation of all Speckle data. Every custom data structure transferred via Speckle MUST inherit from it. Key properties:

| Property | Type | Description |
|----------|------|-------------|
| `id` | string | Unique hash generated from serialized content |
| `applicationId` | string | Optional secondary identity from host application |
| `speckle_type` | string | Type discriminator (assembly name + inheritance chain) |
| `totalChildrenCount` | int | Total number of detachable child objects |
| `units` | string | Unit system for the object |

**Dynamic Properties**: Base objects support arbitrary properties added at runtime:

```python
from specklepy.objects import Base

obj = Base()
obj.name = "my wall"
obj["custom_param"] = 42
obj["@detached_child"] = another_base_object  # @ prefix = detached
```

**Detaching with @ Prefix**: Prepending `@` to a property name causes that property to be stored as a separate reference rather than inline. This prevents duplication when multiple objects reference the same data and is critical for performance with large models.

**Display Values**: Objects include `displayValue` properties for rendering in applications that lack native support for the object type:

```python
# displayValue contains Mesh objects for visual representation
wall.displayValue = [mesh1, mesh2]
```

**Collections**: The `Collection` type represents hierarchical groupings with `name`, `collectionType`, and `elements` properties. Collections MUST form a directed tree (no cycles), unlike regular Speckle objects which may form directed acyclic graphs.

### 1.4 Hashing and Immutability

Objects are immutable — changing any property creates a new identity (new hash). The SDK automatically computes hashes during serialization. This means:

- Two objects with identical content ALWAYS have the same hash
- Modifying a property ALWAYS produces a different hash
- Hashes cascade: changing a child object changes the parent's hash

---

## 2. Speckle Transports

### 2.1 Transport Types

| Transport | Use Case | Notes |
|-----------|----------|-------|
| **ServerTransport** | Cloud storage via Speckle Server | Primary production transport |
| **SQLiteTransport** | Local database cache | Default local cache, customizable path |
| **MemoryTransport** | In-memory storage | For testing, serverless, containers |
| **DiskTransport** | File-based persistence | For offline workflows |
| **MongoDBTransport** | NoSQL database | For custom deployments |

### 2.2 Send/Receive Pattern

Sending serializes objects and writes them to one or more transports simultaneously:

```csharp
// C# - Send to multiple transports in parallel
var objectId = await Operations.Send(
    myData,
    transports: new ITransport[] {
        new ServerTransport(account, streamId),
        new SQLiteTransport(basePath: @"./local-cache"),
        new MemoryTransport()
    }
);
```

Receiving reads from a remote transport with automatic local caching:

```csharp
// C# - Receive with remote source
var myData = await Operations.Receive(
    objectId,
    remoteTransport: new ServerTransport(account, streamId)
);
```

### 2.3 Custom Local Storage

For project-specific storage without polluting the global cache:

```csharp
var customLocalTransport = new SQLiteTransport(
    basePath: @"{localFolderPath}/.speckle"
);

var objectId = await Operations.Send(
    myData,
    transports: new ITransport[] { customLocalTransport },
    useDefaultCache: false
);
```

---

## 3. SpecklePy — Python SDK

### 3.1 Installation and Compatibility

SpecklePy v3 requires Python 3.10+. Install via pip:

```bash
pip install specklepy
```

Local account data is stored at OS-specific paths:
- **Windows**: `%APPDATA%\Speckle` or `<USER>\AppData\Roaming\Speckle`
- **Linux**: `$XDG_DATA_HOME` or `~/.local/share/Speckle`
- **macOS**: `~/.config/Speckle`

### 3.2 Authentication

```python
from specklepy.api.client import SpeckleClient
from specklepy.api.credentials import get_default_account

client = SpeckleClient(host="https://app.speckle.systems")
account = get_default_account()
client.authenticate_with_account(account)

# Alternative: token-based authentication
# client.authenticate_with_token("YOUR_PERSONAL_ACCESS_TOKEN")
```

### 3.3 Stream/Project Operations

```python
# Create a new project (stream)
new_stream_id = client.stream.create(name="AEC Integration Test")

# Get project details
stream = client.stream.get(id=new_stream_id)

# List all projects
streams = client.stream.list()

# Search projects
results = client.stream.search("foundation")
```

### 3.4 Branch/Model and Commit/Version Operations

```python
# List branches (models)
branches = client.branch.list("stream_id")

# Create a branch
branch_id = client.branch.create("stream_id", "structural-model", "Structural elements")

# Get a branch
branch = client.branch.get("stream_id", "structural-model")

# List commits (versions)
commits = client.commit.list("stream_id")

# Get a specific commit
commit = client.commit.get("stream_id", "commit_id")
```

### 3.5 Sending Objects

```python
from specklepy.objects import Base
from specklepy.objects.geometry import Point
from specklepy.transports.server import ServerTransport
from specklepy.api import operations

# Create a custom object
class Block(Base):
    length: float
    width: float
    height: float
    origin: Point = None

    def __init__(self, length=1.0, width=1.0, height=1.0, origin=Point(), **kwargs):
        super().__init__(**kwargs)
        self.add_detachable_attrs({"origin"})
        self.length = length
        self.width = width
        self.height = height
        self.origin = origin

# Send to server
block = Block(length=2.0, height=4.0)
transport = ServerTransport(client=client, stream_id=new_stream_id)

object_hash = operations.send(base=block, transports=[transport])

# Create a commit (version) referencing the object
commit_id = client.commit.create(
    stream_id=new_stream_id,
    object_id=object_hash,
    message="Structural block from Python"
)
```

### 3.6 Receiving Objects

```python
# Receive by object hash
received = operations.receive(obj_id=object_hash, remote_transport=transport)

# Receive via StreamWrapper (from a Speckle URL)
from specklepy.api.wrapper import StreamWrapper

wrapper = StreamWrapper("https://app.speckle.systems/streams/abc123/commits/def456")
client = wrapper.get_client()
transport = wrapper.get_transport()

commit = client.commit.get(wrapper.stream_id, wrapper.commit_id)
received = operations.receive(obj_id=commit.referencedObject, remote_transport=transport)
```

### 3.7 Version Creation (v3 API)

```python
from specklepy.core.api.inputs.version_inputs import CreateVersionInput

version_input = CreateVersionInput(
    project_id=project_id,
    model_id=model_id,
    object_id=object_id,
    message="Updated structural model"
)
version = client.version.create(version_input)
```

### 3.8 Memory Transport for Testing

```python
from specklepy.transports.memory import MemoryTransport

transport = MemoryTransport()
base_obj = Base()
base_obj.name = "test"

obj_hash = operations.send(base=base_obj, transports=[transport])
saved = transport.objects  # dict: {hash: serialized_json}

received = operations.receive(obj_id=obj_hash, remote_transport=transport)
```

### 3.9 Serialization Utilities

```python
from specklepy.serialization.base_object_serializer import BaseObjectSerializer

serializer = BaseObjectSerializer()
hash_val, obj_dict = serializer.traverse_base(base_obj)
hash_val, serialized = serializer.write_json(base_obj)
deserialized = serializer.read_json(serialized)
```

### 3.10 BIM Data Patterns (v3)

In Speckle v3, BIM elements are represented as generic `DataObject` containers rather than typed classes. BIM semantics come from the properties dictionary, not from typed classes.

**CRITICAL**: In v3, do NOT check `speckle_type` — it is ALWAYS `Objects.Data.DataObject`. Instead, inspect properties:

```python
# Check properties for BIM semantics
if obj.properties.get("category") == "Walls":
    print(f"Found a wall: {obj.name}")

# Access Revit parameters (nested structure)
params = obj.properties.get("parameters", {})
type_params = params.get("Type Parameters", {})
instance_params = params.get("Instance Parameters", {})
```

**Display Values**: Geometry is stored in `displayValue` as Mesh objects — these are the atomic selectable items in the Speckle viewer.

**Proxy Collections**: Organizational hierarchies are stored at the root level:
- `levelProxies` — building levels
- `colorProxies` — color grouping
- `groupProxies` — named groups
- `renderMaterialProxies` — material assignments
- `instanceDefinitionProxies` — instance/definition pattern

Each proxy contains `applicationId` lists requiring a two-step lookup: first build an index, then resolve references.

---

## 4. Speckle Blender Connector

### 4.1 Capabilities

The Speckle Blender connector enables bidirectional data exchange between Blender and other AEC tools via the Speckle server. Key supported conversions:

| Blender Type | Speckle Type | Direction |
|-------------|-------------|-----------|
| Mesh | Mesh | Send + Receive |
| Curve (Bezier, NURBS) | Curve/Polycurve | Send + Receive |
| Empty / Collection | Collection | Send + Receive |
| Material | RenderMaterial | Send + Receive |

### 4.2 IFC Data Through Speckle

The IFC-to-Blender workflow via Speckle works as follows:

1. Upload an IFC file directly to Speckle (via the web interface or another connector)
2. Receive the model in Blender using the Speckle connector
3. Original materials from the IFC are automatically imported
4. Models are organized by building levels matching the IFC structure

**Known Limitation — Bonsai Compatibility**: There is currently limited support for Bonsai (formerly BlenderBIM). The workflow is one-directional: you can open an IFC using Bonsai, then use the Speckle connector to send to Speckle. However, the reverse — receiving Speckle data and having it recognized as Bonsai/IFC data — does NOT work reliably. Bonsai's native IFC object model is richer than what Speckle's Base objects can represent.

**What Transfers via Speckle IFC-to-Blender**:
- Mesh geometry (triangulated)
- Material assignments (basic properties, not full PBR)
- Object hierarchy matching IFC spatial structure
- Basic IFC property sets (as dynamic properties on Base objects)

**What Does NOT Transfer**:
- Full IFC relationships (IfcRelAggregates, IfcRelContainedInSpatialStructure details)
- Parametric definitions (only display mesh)
- Bonsai-specific metadata and IFC editing capability
- Detailed property set structure (flattened)

### 4.3 Collection Mapping

Blender collections map to Speckle Collections. The hierarchy is preserved:
- Top-level collection = root Speckle Collection
- Nested collections = child Collections with `collectionType` metadata
- Objects within collections = `elements` list entries

### 4.4 Material Handling

Materials are sent as `RenderMaterial` objects with basic properties (diffuse color, opacity, metalness, roughness). Complex Blender shader node trees are NOT serialized — only the first principled BSDF node's basic parameters are extracted. On receive, materials are recreated as simple Principled BSDF setups.

---

## 5. Speckle Revit Connector

### 5.1 Version Support

| Revit Version | .NET Framework | WebView | Status |
|--------------|---------------|---------|--------|
| 2022-2024 | .NET Framework 4.8 | CefSharp | Supported |
| 2025-2026 | .NET 8.0 | WebView2 | Supported |

### 5.2 Send Pipeline (Revit Elements to Speckle)

The send pipeline processes Revit elements through these stages:

1. **Element Selection**: By view, by selection set, by category filter, or by Revit workset
2. **Element Unpacking**: Groups and complex elements (curtain walls, stairs) are expanded into atomic objects
3. **Conversion**: `RevitRootObjectBuilder` converts each Revit element to a Speckle Base object
4. **Caching**: `RevitToSpeckleCacheSingleton` prevents redundant conversions of identical elements
5. **Transport**: Serialized objects sent to selected transports

**Parameter Extraction**: ALL Revit parameters (both Type and Instance) are nested inside a `parameters` property. Parameters are stored using their internal Revit names, organized by category:

```json
{
  "parameters": {
    "Type Parameters": {
      "Width": { "value": 200, "units": "mm", "isShared": false },
      "Fire Rating": { "value": "1HR", "units": null, "isShared": true }
    },
    "Instance Parameters": {
      "Base Offset": { "value": 0, "units": "mm" },
      "Top Constraint": { "value": "Level 2" }
    }
  }
}
```

### 5.3 Receive Pipeline (Speckle to Revit)

The receive pipeline creates Revit elements from Speckle objects:

1. **Object Reception**: Download and deserialize from transport
2. **Type Mapping**: Match Speckle objects to Revit families/types by Category, Family, and Type properties
3. **DirectShape Creation**: Primary geometry container for received objects
4. **Material Baking**: `RevitMaterialBaker` applies materials and colors
5. **Group Creation**: `RevitGroupBaker` creates Revit groups for organizational structure
6. **Transaction Commit**: All changes wrapped in Revit API transactions

**Type Mapping Options**:
- **Automatic**: Speckle attempts to match Category + Family + Type automatically
- **Manual**: User presented with a mapping table to manually assign Revit types

**Tessellation Quality**: Controlled via DetailLevel setting (Low/Medium/High) affecting mesh density of received geometry.

### 5.4 Round-Trip Data Integrity

**What SURVIVES the Revit → Speckle → Revit round-trip**:
- Geometry (received as DirectShapes)
- Parameter values (both Type and Instance, when not null/empty)
- Material assignments (basic properties)
- Object grouping and hierarchy
- ApplicationId tracking (enables update-in-place on subsequent receives)

**What is LOST or DEGRADED**:
- **Parametric intelligence**: Elements return as DirectShapes, NOT as editable Revit families. A wall sent from Revit comes back as a DirectShape resembling a wall, not an editable Wall element.
- **System families**: Walls, floors, roofs lose their system-family behavior
- **Hosted elements**: Relationships between hosts (walls) and hosted elements (doors, windows) are flattened
- **Detailed schedules**: Schedule-specific parameter grouping is lost
- **Workset assignments**: Not preserved across round-trips
- **Design options**: Not tracked through Speckle

**Update Behavior**: When receiving a second time from the same source, the connector checks `applicationId` values. If a received element matches a previously received `applicationId`, the connector updates the existing element rather than creating a duplicate. This preserves Dimensions, ElementIds, and annotations attached to previously received elements.

### 5.5 Linked Model Support

The connector supports sending linked Revit models. Each linked model creates a separate `DocumentToConvert` context with a transformation matrix. ApplicationId values are modified with a transform hash to distinguish instances of the same linked model at different positions.

### 5.6 Reference Point Settings

Send and receive operations support three reference point options:
- **Internal Origin**: Revit's internal coordinate origin
- **Project Base Point**: The project-defined base point
- **Survey Point**: The survey-defined coordinate origin

This setting is critical for maintaining spatial alignment when exchanging data between Revit and GIS/geospatial tools.

---

## 6. QGIS-BIM/IFC Integration

### 6.1 Overview of Approaches

There are three primary approaches to integrating IFC/BIM data with QGIS, each with distinct trade-offs:

| Approach | Flexibility | Geometry Support | Effort |
|----------|------------|-----------------|--------|
| GDAL/OGR (ogr2ogr) | Low | Very Limited | Low |
| IfcOpenShell + PyQGIS | High | Full 3D | High |
| Speckle QGIS Connector | Medium | 2D + Basic 3D | Medium |

### 6.2 Approach A: GDAL IFC Driver (ogr2ogr)

**Status**: GDAL/OGR does NOT have a native IFC driver as of GDAL 3.9. The IFC format is not listed among GDAL's supported vector drivers. Earlier documentation references were either experimental or community-contributed and never reached stable release.

**Verified**: The GDAL vector driver list (https://gdal.org/en/stable/drivers/vector/index.html) does NOT include an IFC entry. The URL `https://gdal.org/en/stable/drivers/vector/ifc.html` returns 404.

**Workaround via ogr2ifc**: The community project [ogr2ifc](https://github.com/Robinini/ogr2ifc) provides GIS-to-BIM conversion using GDAL/OGR for reading GIS formats and IfcOpenShell for writing IFC. This is the reverse direction (GIS → IFC) rather than IFC → GIS.

**Implication**: GDAL cannot be used directly to read IFC files into QGIS. Alternative approaches (IfcOpenShell or Speckle) are ALWAYS required.

### 6.3 Approach B: IfcOpenShell + PyQGIS (Recommended)

This is the most flexible approach, providing full control over geometry extraction, CRS assignment, and attribute mapping.

#### 6.3.1 IfcOpenShell Geometry Iterator

IfcOpenShell (v0.8.4) provides a geometry iterator for efficient batch processing:

```python
import ifcopenshell
import ifcopenshell.geom

# Open IFC file
ifc_file = ifcopenshell.open("building.ifc")

# Configure geometry settings
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)  # Apply transformations
settings.set(settings.WELD_VERTICES, True)      # Merge duplicate vertices

# Use geometry iterator for batch processing (ALWAYS prefer over create_shape)
iterator = ifcopenshell.geom.iterator(settings, ifc_file, multiprocessing.cpu_count())

if iterator.initialize():
    while True:
        shape = iterator.get()
        element = ifc_file.by_id(shape.id)

        # Get geometry data
        geometry = shape.geometry
        verts = geometry.verts      # Flat list: [x1,y1,z1, x2,y2,z2, ...]
        faces = geometry.faces      # Flat list: [i1,i2,i3, i4,i5,i6, ...]

        # Process vertices into coordinate tuples
        coords = [(verts[i], verts[i+1], verts[i+2])
                  for i in range(0, len(verts), 3)]

        if not iterator.next():
            break
```

**IMPORTANT**: ALWAYS use the geometry iterator for batch processing. Using `create_shape()` individually per element is significantly less efficient.

#### 6.3.2 Footprint Extraction with Shapely

To extract 2D footprints for GIS overlay:

```python
import ifcopenshell
import ifcopenshell.geom
from shapely.geometry import Polygon, MultiPolygon
from shapely.ops import unary_union
import geopandas as gpd

def extract_footprints(ifc_path, target_epsg=28992):
    """Extract 2D footprints from IFC elements and create a GeoDataFrame."""
    ifc_file = ifcopenshell.open(ifc_path)

    settings = ifcopenshell.geom.settings()
    settings.set(settings.USE_WORLD_COORDS, True)

    features = []

    # Process walls, slabs, and other building elements
    for element in ifc_file.by_type("IfcBuildingElement"):
        try:
            shape = ifcopenshell.geom.create_shape(settings, element)
            verts = shape.geometry.verts
            faces = shape.geometry.faces

            # Extract XY coordinates (drop Z for footprint)
            coords_2d = [(verts[i], verts[i+1])
                         for i in range(0, len(verts), 3)]

            if len(coords_2d) >= 3:
                # Create convex hull as simplified footprint
                from shapely.geometry import MultiPoint
                footprint = MultiPoint(coords_2d).convex_hull

                features.append({
                    "geometry": footprint,
                    "ifc_guid": element.GlobalId,
                    "ifc_type": element.is_a(),
                    "name": getattr(element, "Name", None),
                })
        except Exception:
            continue  # Skip elements without geometry

    gdf = gpd.GeoDataFrame(features, crs=f"EPSG:{target_epsg}")
    return gdf

# Export to GeoPackage for QGIS
gdf = extract_footprints("building.ifc", target_epsg=28992)
gdf.to_file("building_footprints.gpkg", driver="GPKG")
```

#### 6.3.3 CRS Assignment and Georeferencing

IFC files store georeferencing in `IfcMapConversion` and `IfcProjectedCRS` entities (IFC4+):

```python
import ifcopenshell

ifc_file = ifcopenshell.open("building.ifc")

# Extract georeferencing from IFC
map_conversion = ifc_file.by_type("IfcMapConversion")
projected_crs = ifc_file.by_type("IfcProjectedCRS")

if map_conversion:
    mc = map_conversion[0]
    easting = mc.Eastings       # In map units (typically meters)
    northing = mc.Northings
    ortho_height = mc.OrthogonalHeight
    x_axis_abscissa = mc.XAxisAbscissa   # Rotation component
    x_axis_ordinate = mc.XAxisOrdinate

if projected_crs:
    crs = projected_crs[0]
    epsg_name = crs.Name           # e.g., "EPSG:28992"
    geodetic_datum = crs.GeodeticDatum
```

**CRITICAL**: IFC project units are often millimeters, while map coordinates (Eastings/Northings) are in meters. ALWAYS verify the unit scaling. When setting georeferencing parameters in Bonsai or other BIM tools, multiply coordinate values by 1000 if the IFC file uses millimeters.

#### 6.3.4 Loading into QGIS via PyQGIS

```python
from qgis.core import (
    QgsVectorLayer, QgsProject, QgsCoordinateReferenceSystem
)

# Load GeoPackage layer into QGIS
layer = QgsVectorLayer("building_footprints.gpkg", "BIM Footprints", "ogr")

if layer.isValid():
    # Set CRS explicitly if not embedded
    crs = QgsCoordinateReferenceSystem("EPSG:28992")
    layer.setCrs(crs)

    QgsProject.instance().addMapLayer(layer)
    print(f"Loaded {layer.featureCount()} BIM elements into QGIS")
else:
    print("Layer failed to load")
```

### 6.4 Approach C: Speckle QGIS Connector

#### 6.4.1 Installation and Setup

The Speckle QGIS connector installs via QGIS Plugin Manager: `Plugins > Manage and install plugins > search "Speckle"`. Supports QGIS 3.28.15 and above.

**NOTE**: As of 2025, the original QGIS connector is marked as a **legacy connector**. Speckle recommends using their newer connectors, though the QGIS legacy connector received updates as recently as March 2025.

#### 6.4.2 Sending QGIS Data to Speckle

The connector allows selecting multiple QGIS layers and sending their geometry + attributes to a Speckle server. Geometry is reprojected based on the project's CRS before sending.

**Supported send geometry types**: Points, Lines, Polylines, Polygons (2D and 2.5D with elevation attributes).

#### 6.4.3 Receiving BIM Data in QGIS

Since Speckle 2.11, the QGIS connector can receive BIM models. Received objects are imported as QGIS layers with:
- Geometry converted to QGIS-compatible types
- Object properties stored in the layer attribute table
- Layers organized in a group named with project/model/version info

#### 6.4.4 CRS Handling

The connector provides three CRS customization approaches:

1. **Offset CRS**: Add X/Y offsets to an existing CRS without changing the working projection
2. **Custom CRS**: Create a Transverse Mercator projection specifying geographic origin coordinates
3. **True North Angle**: Adjust orientation for alignment with non-GIS coordinate systems

#### 6.4.5 3D Transformations (v2.14+)

When "Apply transformations on Send" is activated:
- DEM (Digital Elevation Model) conversion to mesh
- Raster-to-mesh texturing
- Polygon extrusion using attribute values for height
- Polygon projection onto 3D terrain surfaces

This enables sending full 3D context maps from QGIS for use in CAD/BIM software.

#### 6.4.6 Limitations

- Geographic CRS with non-linear units are treated as meters in receiving software, causing coordinate misinterpretation
- Limited geometry type support for receiving (Point, Line, Polyline, Arc, Circle, Polycurve)
- No native IFC entity type preservation — BIM elements arrive as generic geometry with attributes
- Legacy connector status means active development has slowed

---

## 7. Common Speckle Integration Failures

### 7.1 Object Conversion Loss

| Data Type | Behavior | Severity |
|-----------|----------|----------|
| Parametric definitions | Lost — only display mesh transferred | HIGH |
| NURBS surfaces | Converted to mesh (tessellated) | MEDIUM |
| Complex materials | Simplified to basic diffuse/opacity/metalness | MEDIUM |
| IFC relationships | Flattened to properties | HIGH |
| Hosted element connections | Lost (doors detached from walls) | HIGH |
| Workset assignments | Not transferred | LOW |
| Design options | Not tracked | LOW |
| Shared parameters | Transferred but may lose shared parameter GUID | MEDIUM |

### 7.2 Relationship Flattening

Speckle's Base object model is a property tree, not a relational database. IFC's rich relationship model (IfcRelAggregates, IfcRelAssociatesMaterial, IfcRelDefinesByProperties) is flattened:

- **Aggregation** (building → storey → space) becomes a Collection hierarchy
- **Material associations** become properties on the element
- **Property sets** become nested dictionaries
- **Type/occurrence relationships** are lost — each occurrence carries a copy of type properties

This means that receiving an IFC model via Speckle and re-exporting does NOT produce a valid, relationship-rich IFC file.

### 7.3 Large Model Performance

- Object decomposition creates many small objects — a 100MB IFC file may produce 500,000+ Speckle objects
- Server upload/download becomes the bottleneck for models > 500MB
- SQLite local cache can grow significantly (multi-GB)
- The Revit connector uses instancing optimization (geometry stored once, proxies for repeated instances) to mitigate this

### 7.4 Version Conflicts

- Speckle v2 and v3 have different data models (streams/branches/commits vs workspaces/projects/models/versions)
- SpecklePy v3 requires Python 3.10+ (breaking change from v2's Python 3.6+ support)
- `speckle_type` semantics changed: v2 uses typed classes, v3 uses generic `DataObject` with property-based semantics
- Connector versions MUST match server version within the same major version

### 7.5 Material Loss Chain

Revit (PBR materials, render appearances) → Speckle (RenderMaterial: diffuse, opacity, metalness, roughness) → Blender (Principled BSDF, basic parameters only)

At each boundary crossing, material fidelity decreases. Texture maps are NOT transferred. Normal maps, displacement maps, and procedural materials are lost entirely.

---

## 8. Common QGIS-BIM Integration Failures

### 8.1 2D Projection of 3D BIM Data

BIM models are inherently 3D. When imported into QGIS (primarily a 2D GIS with limited 3D support):

- **Vertical overlap**: Multi-storey elements project onto the same 2D footprint, creating overlapping polygons
- **Z-coordinate loss**: Standard GIS formats (Shapefile) drop Z values; use GeoPackage or PostGIS to preserve them
- **Floor-by-floor filtering**: ALWAYS filter by building storey before projecting to 2D to avoid overlap

### 8.2 CRS Misalignment

The most common and most damaging failure in BIM-GIS integration:

| Issue | Cause | Solution |
|-------|-------|----------|
| Model "in the ocean" | IfcMapConversion not applied to geometry | Apply Eastings/Northings transformation explicitly |
| Wrong hemisphere | Sign error in coordinates | Verify EPSG code matches hemisphere |
| Off by factor 1000 | mm vs m unit mismatch | Check IFC project units and scale accordingly |
| Rotated model | XAxisAbscissa/Ordinate not applied | Apply rotation from IfcMapConversion |
| Wrong datum | Incorrect EPSG assignment | Verify with known survey points |

### 8.3 Attribute Mapping Challenges

- IFC property sets use hierarchical naming (`Pset_WallCommon.FireRating`) while GIS expects flat column names
- Property set names may exceed GIS field name limits (10 chars for Shapefile, 256 for GeoPackage)
- Quantity sets (`Qto_WallBaseQuantities`) contain calculated values that may not match actual geometry in simplified GIS representation
- Enumerated values in IFC (e.g., `IfcWallTypeEnum.STANDARD`) need string conversion

### 8.4 Geometry Simplification

- BIM curved walls (IfcExtrudedAreaSolid with IfcCircleProfileDef) become tessellated polygons
- Boolean operations (wall openings) must be resolved before export — IfcOpenShell handles this automatically
- Thin elements (steel profiles, rebar) may collapse to zero-area footprints in 2D projection
- Tolerance differences: BIM typically works at mm precision, GIS at cm/m precision

### 8.5 Performance with Large Datasets

- IFC files > 100MB can take minutes to process with IfcOpenShell geometry iterator
- Enable multiprocessing: `ifcopenshell.geom.iterator(settings, ifc_file, num_cores)`
- GeoPackage files with 100,000+ features need spatial indexing for acceptable QGIS performance
- Consider filtering by IFC entity type (IfcWall, IfcSlab) rather than processing all IfcBuildingElement subtypes

---

## 9. Integration Patterns Summary

### 9.1 Recommended Pattern: Revit → Speckle → QGIS

```
Revit Model
    ↓ (Speckle Revit Connector)
Speckle Server
    ↓ (SpecklePy or Speckle QGIS Connector)
QGIS Layers (with attributes from Revit parameters)
```

**Pros**: Automated pipeline, versioned, accessible to multiple tools
**Cons**: Parametric data lost, relationship flattening, material simplification

### 9.2 Recommended Pattern: IFC → IfcOpenShell → GeoPackage → QGIS

```
IFC File
    ↓ (IfcOpenShell + Shapely)
GeoDataFrame (with CRS from IfcMapConversion)
    ↓ (GeoPandas .to_file())
GeoPackage
    ↓ (native QGIS support)
QGIS Layer
```

**Pros**: Full control, preserves more IFC metadata, handles georeferencing explicitly
**Cons**: Custom scripting required, no automatic versioning

### 9.3 Recommended Pattern: Blender/Bonsai → Speckle → Other Tools

```
Bonsai IFC Model in Blender
    ↓ (Speckle Blender Connector — send only)
Speckle Server
    ↓ (Speckle Connectors for target tools)
Revit / QGIS / Web Viewer
```

**Pros**: Single source of truth in Blender/Bonsai, multi-target distribution
**Cons**: One-way only (receiving back into Bonsai loses IFC intelligence), material degradation

---

## 10. Version Matrix

| Technology | Version Verified | Notes |
|------------|-----------------|-------|
| SpecklePy | v3 (requires Python 3.10+) | Breaking changes from v2 |
| Speckle Server | 2.x / transitioning to 3.x | v3 renames streams→projects, branches→models |
| Speckle Blender Connector | Latest (2025) | Matches Blender 3.x/4.x |
| Speckle Revit Connector | Revit 2022-2026 | .NET 8 for Revit 2025+ |
| Speckle QGIS Connector | QGIS 3.28.15+ | Legacy status, updated March 2025 |
| IfcOpenShell | 0.8.4 | Geometry iterator, Python bindings |
| GDAL | 3.9 | NO native IFC driver |
| Shapely | 2.x | For footprint extraction |
| GeoPandas | 0.14+ | For GeoPackage export |

---

## 11. Verification Notes

| Claim | Status | Source |
|-------|--------|--------|
| Speckle Base object hashing is immutable | **Verified** | speckle.guide/dev/base.html |
| SpecklePy v3 requires Python 3.10+ | **Verified** | docs.speckle.systems |
| @ prefix detaches properties | **Verified** | speckle.guide/dev/base.html, py-examples |
| GDAL has NO IFC driver | **Verified** | gdal.org returns 404 for IFC driver page |
| Speckle QGIS connector is legacy | **Verified** | docs.speckle.systems/legacy/user/qgis |
| Revit connector supports 2022-2026 | **Verified** | deepwiki.com Revit connector docs |
| Bonsai→Speckle is one-way only | **Verified** | speckle.community discussion |
| DirectShapes are primary receive container in Revit | **Verified** | deepwiki.com Revit connector architecture |
| v3 uses DataObject instead of typed classes | **Verified** | docs.speckle.systems BIM data patterns |
| IfcOpenShell geometry iterator supports multiprocessing | **Verified** | docs.ifcopenshell.org |
| IFC project units often mm, map coords in m | **Inferred** | Common in Dutch/European BIM practice, confirmed in OSArch discussion |
| Material loss chain (Revit→Speckle→Blender) | **Inferred** | Based on RenderMaterial property limitations documented in Speckle |
| 100MB IFC = 500k+ Speckle objects | **Inferred** | Based on decomposition architecture, not measured |

---

## Sources

- [Speckle Architecture (Legacy)](https://speckle.guide/dev/architecture.html)
- [Speckle Base Object (Legacy)](https://speckle.guide/dev/base.html)
- [Speckle Transports (Legacy)](https://speckle.guide/dev/transports.html)
- [SpecklePy Examples (Legacy)](https://speckle.guide/dev/py-examples.html)
- [SpecklePy Introduction (v3)](https://docs.speckle.systems/developers/sdks/python/introduction)
- [BIM Data Patterns (v3)](https://docs.speckle.systems/developers/sdks/python/guides/bim-data-patterns)
- [Speckle QGIS Connector (Legacy)](https://docs.speckle.systems/legacy/user/qgis)
- [Speckle Revit Connector (DeepWiki)](https://deepwiki.com/specklesystems/speckle-sharp-connectors/3-revit-connector)
- [Speckle QGIS Plugin Repository](https://plugins.qgis.org/plugins/speckle-qgis/)
- [Speckle Community: IFC data into Blender](https://speckle.community/t/ifc-data-into-blender/21048)
- [OSArch: IFC to QGIS GeoPackage](https://community.osarch.org/discussion/3037/ifc-to-qgis-geopackage)
- [IfcOpenShell Geometry Iterator](https://docs.ifcopenshell.org/ifcopenshell/geometry_iterator.html)
- [IfcOpenShell Documentation](https://docs.ifcopenshell.org/)
- [ogr2ifc (GIS to BIM)](https://github.com/Robinini/ogr2ifc)
- [SpecklePy on GitHub](https://github.com/specklesystems/specklepy)
- [Speckle for QGIS](https://speckle.systems/connectors/qgis)
