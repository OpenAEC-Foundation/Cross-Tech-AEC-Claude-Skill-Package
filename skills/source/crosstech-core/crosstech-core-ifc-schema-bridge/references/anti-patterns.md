# crosstech-core-ifc-schema-bridge: Anti-Patterns

These are confirmed error patterns when mapping IFC data across technology boundaries.
Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY.

Sources:
- vooronderzoek-ifc-ecosystem.md (sections 6.1–6.8)
- https://docs.ifcopenshell.org/
- https://github.com/ThatOpen/engine_web-ifc

---

## AP-001: Querying Only Occurrence Properties (Missing Type Properties)

**WHY this is wrong**: IFC splits properties between occurrence-level (`IfcWall`) and type-level (`IfcWallType`). Properties on the type are shared by all instances. Querying only the occurrence misses shared properties — typically 30-70% of all properties are on the type.

```python
# WRONG — only gets occurrence-level properties
psets = {}
for rel in wall.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        if pset.is_a("IfcPropertySet"):
            psets[pset.Name] = {p.Name: p.NominalValue for p in pset.HasProperties}
# MISSING: type-level properties from IfcWallType
```

```python
# CORRECT — gets properties from BOTH occurrence AND type
import ifcopenshell.util.element
psets = ifcopenshell.util.element.get_psets(wall)
# This function ALWAYS includes type-level properties automatically
```

ALWAYS use `ifcopenshell.util.element.get_psets()` which merges occurrence and type properties. NEVER manually traverse `IsDefinedBy` unless you explicitly handle both levels.

---

## AP-002: Assuming IFC Units Are Meters

**WHY this is wrong**: IFC files can use any unit system (millimeters, centimeters, meters, inches, feet). If the consuming tool assumes meters but the file uses millimeters, all dimensions appear 1000x too large. This causes models to be placed kilometers away from the origin or appear at absurd scales.

```python
# WRONG — assumes meters
wall_length = psets["Qto_WallBaseQuantities"]["Length"]
print(f"Wall is {wall_length} meters long")  # WRONG if file uses millimeters

# WRONG — assumes geometry coordinates are in meters
shape = ifcopenshell.geom.create_shape(settings, wall)
x, y, z = shape.geometry.verts[0:3]  # Could be in mm, cm, m, inches, feet
```

```python
# CORRECT — ALWAYS apply unit scale
import ifcopenshell.util.unit
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
# unit_scale converts file units to SI meters

wall_length_meters = psets["Qto_WallBaseQuantities"]["Length"] * unit_scale
x_meters = shape.geometry.verts[0] * unit_scale
```

ALWAYS call `ifcopenshell.util.unit.calculate_unit_scale(model)` before using any numeric values from an IFC file.

---

## AP-003: Overwriting IFC Geometry with Tessellated Versions

**WHY this is wrong**: IFC stores parametric geometry (extruded solids, swept solids, BREP, CSG). Tessellation converts this to triangles, which is a one-way operation. Replacing parametric geometry with tessellated triangles causes: 10-100x file size increase, loss of curved surface accuracy, failure of boolean operations, inability to edit dimensions parametrically.

```python
# WRONG — extracts tessellated mesh and writes it back as IFC geometry
shape = ifcopenshell.geom.create_shape(settings, wall)
verts = shape.geometry.verts
faces = shape.geometry.faces
# ... creates IfcTriangulatedFaceSet from tessellated data ...
# ... assigns it back to the wall, replacing the original parametric geometry ...
```

```python
# CORRECT — keep original IFC file as source of truth
# Use tessellated geometry ONLY for display in target tools
# NEVER write tessellated geometry back to the IFC file
shape = ifcopenshell.geom.create_shape(settings, wall)
# Use verts/faces for Three.js, Blender display mesh, etc.
# The original IFC geometry remains untouched in the file
```

NEVER replace parametric IFC geometry with tessellated versions. ALWAYS keep the IFC file as the single source of truth.

---

## AP-004: Ignoring Schema Version Differences

**WHY this is wrong**: IFC4X3 adds ~150 new entity types for infrastructure (roads, railways, bridges, alignments). An IFC4-only tool silently drops these entities — no error is raised, the data simply disappears. This causes infrastructure models to appear empty or missing critical elements.

```python
# WRONG — processes file without checking schema
model = ifcopenshell.open("model.ifc")
walls = model.by_type("IfcWall")  # Works, but misses infrastructure entities
# If this is an IFC4X3 road model, IfcRoad, IfcAlignment, etc. are ignored
```

```python
# CORRECT — ALWAYS check schema version first
model = ifcopenshell.open("model.ifc")
schema = model.schema

if schema == "IFC4X3":
    # Process infrastructure entities
    alignments = model.by_type("IfcAlignment")
    roads = model.by_type("IfcRoad")
    if alignments:
        print(f"Found {len(alignments)} alignment(s) — IFC4X3 infrastructure model")
elif schema == "IFC4":
    print("IFC4 building model — no infrastructure entities available")
elif schema == "IFC2X3":
    # Handle legacy differences
    # IfcWallStandardCase exists in IFC2X3 but not in IFC4/IFC4X3
    walls = model.by_type("IfcWallStandardCase") + model.by_type("IfcWall")
```

```javascript
// web-ifc — ALWAYS check schema
const schema = ifcApi.GetModelSchema(modelID);
if (schema === "IFC4X3") {
  console.warn("IFC4X3 model — verify that all entity types are supported");
}
```

ALWAYS check `model.schema` (IfcOpenShell) or `ifcApi.GetModelSchema(modelID)` (web-ifc) as the first operation after opening a file.

---

## AP-005: Storing IFC Properties in Three.js Scene Graph

**WHY this is wrong**: Three.js scene graphs are optimized for rendering, not data storage. Storing IFC properties as `userData` on every `Object3D` bloats memory, breaks garbage collection (properties hold references to large objects), and makes querying slow (must traverse entire scene). ThatOpen Components deliberately separates geometry (Fragments) from semantics (web-ifc queries).

```javascript
// WRONG — storing all properties on Three.js objects
const mesh = new THREE.Mesh(geometry, material);
mesh.userData.properties = getAllProperties(expressID);  // Large nested object
mesh.userData.relationships = getAllRelationships(expressID);  // More data
scene.add(mesh);
// Result: scene uses 3-10x more memory than necessary
```

```javascript
// CORRECT — maintain parallel data channel
const mesh = new THREE.Mesh(geometry, material);
mesh.userData.expressID = expressID;  // Store ONLY the ID
scene.add(mesh);

// Query properties on demand via web-ifc
function getProperties(expressID: number) {
  return ifcApi.GetLine(modelID, expressID, true);
}

// Or pre-extract to a Map for fast lookup
const propertyMap = new Map<number, any>();
// ... populate during initial load ...
```

NEVER store full IFC property data on Three.js objects. ALWAYS use `expressID` as a lightweight key and query data from web-ifc on demand.

---

## AP-006: Flattening IFC Relationships to a Simple Tree

**WHY this is wrong**: IFC relationships are many-to-many objectified entities. A single wall can be: contained in a storey (IfcRelContainedInSpatialStructure), part of a group (IfcRelAssignsToGroup), associated with a material (IfcRelAssociatesMaterial), defined by a type (IfcRelDefinesByType), and voided by openings (IfcRelVoidsElement) — ALL simultaneously. Flattening to a parent-child tree loses all but one of these relationships.

```javascript
// WRONG — converts IFC to a simple parent-child tree
const tree = {
  project: {
    children: [
      { site: { children: [
        { building: { children: [
          { storey: { children: [
            { wall: {} },  // Lost: type, material, properties, openings, groups
          ]}}
        ]}}
      ]}}
    ]
  }
};
```

```javascript
// CORRECT — maintain multiple relationship indices
const spatialTree = {};  // IfcRelAggregates + IfcRelContainedInSpatialStructure
const typeMap = new Map();  // expressID → typeExpressID (IfcRelDefinesByType)
const materialMap = new Map();  // expressID → materialExpressID (IfcRelAssociatesMaterial)
const groupMap = new Map();  // expressID → groupExpressIDs (IfcRelAssignsToGroup)
const voidMap = new Map();  // expressID → openingExpressIDs (IfcRelVoidsElement)
const propertyMap = new Map();  // expressID → psetExpressIDs (IfcRelDefinesByProperties)
```

NEVER reduce IFC relationships to a single hierarchy. ALWAYS maintain separate indices for each relationship type.

---

## AP-007: Using Blender Import Without Bonsai for BIM Data

**WHY this is wrong**: Blender's built-in IFC import (without Bonsai) creates mesh objects with basic names but loses ALL semantic data: properties, relationships, types, spatial structure, and material layer sets. The result is geometry with no BIM intelligence.

```python
# WRONG — import IFC without Bonsai
bpy.ops.import_scene.ifc(filepath="model.ifc")  # Basic geometry import
# Result: mesh objects named "IfcWall_001" but NO properties, NO relationships, NO type info
```

```python
# CORRECT — use Bonsai for IFC import
# Bonsai maintains a parallel IFC data structure alongside Blender objects
# Install Bonsai extension, then:
import blenderbim.tool as tool
# Bonsai preserves: properties, relationships, types, spatial structure
# The IFC file remains the source of truth
```

NEVER import IFC files into Blender without the Bonsai extension if you need BIM data. ALWAYS use Bonsai for any IFC workflow in Blender.

---

## AP-008: Comparing Property Values Without Encoding Normalization

**WHY this is wrong**: IFC STEP files use ISO-8859-1 encoding with special escape sequences for extended characters. The string "Wohnfläche" (German for living area) might be stored as `'Wohnfl\X2\00E4\X0\che'` in the raw file. Different parsers handle decoding differently — comparing raw strings across tools produces false mismatches.

```python
# WRONG — comparing raw property values from different sources
ios_value = ifcopenshell_wall.Name  # "Wohnfläche" (decoded by IfcOpenShell)
webifc_value = raw_ifc_line  # "Wohnfl\X2\00E4\X0\che" (raw STEP encoding)
assert ios_value == webifc_value  # FAILS — different encodings
```

```python
# CORRECT — normalize before comparing
# IfcOpenShell automatically decodes IFC string encoding
ios_value = ifcopenshell_wall.Name  # Already decoded: "Wohnfläche"

# web-ifc also decodes in recent versions
# For raw STEP files, use IfcOpenShell's decoding or implement ISO 10303-21 Part 21 decoding
```

ALWAYS normalize string encoding before comparing property values across tools. IfcOpenShell and recent web-ifc versions handle decoding automatically. NEVER compare raw STEP-encoded strings with decoded strings.

---

## AP-009: Processing Large IFC Files Without Streaming

**WHY this is wrong**: Loading all geometry from a 500MB+ IFC file into memory at once causes out-of-memory crashes in both web-ifc (2GB WASM limit) and IfcOpenShell (Python memory pressure). Large BIM models with thousands of elements produce millions of triangles.

```python
# WRONG — loads all geometry into a list
all_shapes = []
for product in model.by_type("IfcProduct"):
    shape = ifcopenshell.geom.create_shape(settings, product)
    all_shapes.append(shape)  # Memory grows unbounded
```

```javascript
// WRONG — gets all flat meshes at once
const allMeshes = [];
const allLines = ifcApi.GetAllLines(modelID);
for (const id of allLines) {
  allMeshes.push(ifcApi.GetFlatMesh(modelID, id));  // WASM memory exhaustion
}
```

```python
# CORRECT — use streaming iterator (IfcOpenShell)
import multiprocessing
iterator = ifcopenshell.geom.iterator(settings, model, multiprocessing.cpu_count())
if iterator.initialize():
    while True:
        shape = iterator.get()
        # Process one shape at a time, then release
        process_shape(shape)
        if not iterator.next():
            break
```

```javascript
// CORRECT — use streaming callback (web-ifc)
ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  // Process one mesh at a time
  processGeometry(flatMesh);
  // Memory is managed by the streaming callback
});
```

ALWAYS use streaming/iterator patterns for geometry processing. NEVER load all geometry into memory at once.

---

## AP-010: Ignoring IfcMapConversion for Geolocation

**WHY this is wrong**: IFC models use a local coordinate system. The real-world position is defined by `IfcMapConversion` (IFC4+) or `IfcSite.RefLatitude/RefLongitude` (IFC2X3). Ignoring this entity places the model at the wrong geographic location, causing misalignment when combining BIM with GIS data or with other BIM models from different origins.

```python
# WRONG — assumes model coordinates are geographic
wall_coords = ifcopenshell.util.placement.get_local_placement(wall.ObjectPlacement)
# These are LOCAL coordinates, not geographic coordinates
latitude = wall_coords[0]  # WRONG — this is a local X coordinate, not latitude
```

```python
# CORRECT — check for IfcMapConversion
map_conversions = model.by_type("IfcMapConversion")
if map_conversions:
    mc = map_conversions[0]
    easting = mc.Eastings
    northing = mc.Northings
    ortho_height = mc.OrthogonalHeight
    x_axis_abscissa = mc.XAxisAbscissa  # Rotation component
    x_axis_ordinate = mc.XAxisOrdinate  # Rotation component
    scale = mc.Scale or 1.0
    print(f"Model origin: E{easting}, N{northing}, H{ortho_height}")
    # The CRS is defined in the linked IfcProjectedCRS
    crs = mc.TargetCRS
    if crs:
        print(f"CRS: {crs.Name}")  # e.g., "EPSG:28992" (Dutch RD)
else:
    print("WARNING: No IfcMapConversion found — model has no georeferencing")
```

ALWAYS check for `IfcMapConversion` before combining IFC data with GIS data. NEVER treat local IFC coordinates as geographic coordinates.
