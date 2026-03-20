# Vooronderzoek: BIM↔GIS Coordinate System Integration

> Research document for the Cross-Tech AEC Skill Package.
> Topic: Coordinate Reference Systems, IFC Georeferencing, and BIM↔GIS data exchange.
> Date: 2026-03-20
> Sources: buildingSMART IFC4/IFC4X3 docs, PROJ/pyproj docs, IfcOpenShell 0.8.x docs, EPSG registry, QGIS docs.

---

## 1. Coordinate Reference Systems (CRS) Fundamentals

A **Coordinate Reference System (CRS)** defines how coordinates map to real-world locations on Earth. Every CRS consists of three components: a **datum** (the reference model of the Earth's shape), a **coordinate system** (the axes and units), and optionally a **projection** (the mathematical transformation from the curved Earth to a flat surface).

### Geographic vs Projected CRS

| Property | Geographic CRS | Projected CRS |
|----------|---------------|---------------|
| Coordinates | Latitude, Longitude (degrees) | Easting, Northing (meters/feet) |
| Shape model | Ellipsoid (e.g., GRS80, WGS84) | Flat plane derived from ellipsoid |
| Units | Decimal degrees | Linear (meters, feet) |
| Distortion | None (represents real shape) | Always present (area, shape, distance, or direction) |
| Use case | GPS, global datasets | Engineering, mapping, construction |

### EPSG Codes

The **EPSG Geodetic Parameter Dataset** (maintained by the International Association of Oil & Gas Producers, IOGP) assigns numeric codes to CRS definitions. These codes are the standard way to unambiguously identify a CRS across all GIS and BIM software.

### Key CRS Definitions for AEC Work

| EPSG Code | Name | Type | Use Case | Units |
|-----------|------|------|----------|-------|
| **4326** | WGS 84 | Geographic 2D | GPS, global web maps, Speckle default | Degrees |
| **28992** | Amersfoort / RD New | Projected | Netherlands cadastre, engineering, topographic mapping | Meters |
| **7415** | Amersfoort / RD New + NAP height | Compound (28992 + 5709) | Netherlands 3D (horizontal + vertical) | Meters |
| **7931** | ETRS89 | Geographic 3D | European framework | Degrees + meters |
| **32631** | WGS 84 / UTM zone 31N | Projected | Western Netherlands, Belgium | Meters |
| **32632** | WGS 84 / UTM zone 32N | Projected | Eastern Netherlands, Germany | Meters |
| **3857** | WGS 84 / Pseudo-Mercator | Projected | Web maps (OpenStreetMap, Google Maps) | Meters |
| **5709** | NAP height | Vertical | Netherlands vertical datum | Meters |

### EPSG:28992 — Amersfoort / RD New (Detail)

This is the most important CRS for Dutch AEC projects. Key parameters:

- **Datum**: Amersfoort (based on Bessel 1841 ellipsoid)
- **Projection**: Oblique Stereographic (Double)
- **Latitude of origin**: 52.15616055555556°
- **Central meridian**: 5.38763888888889°
- **Scale factor**: 0.9999079
- **False easting**: 155,000 m
- **False northing**: 463,000 m
- **TOWGS84 parameters**: 565.4171, 50.3319, 465.5524, 1.9342, -1.6677, 9.1019, 4.0725

The TOWGS84 parameters define a 7-parameter Helmert transformation (3 translations in meters, 3 rotations in arc-seconds, 1 scale factor in ppm) to convert from the Amersfoort datum to WGS84. This yields approximately 1-meter accuracy. For sub-meter accuracy in the Netherlands, use the official RDNAPTRANS grid-based transformation (since 2000).

### Datum Transformations — Lossy vs Lossless

| Transformation Type | Lossy? | Typical Accuracy | Example |
|---------------------|--------|-----------------|---------|
| Between same datum, different projection | Lossless | Exact (float precision only) | UTM 31N ↔ UTM 32N (both WGS84) |
| 7-parameter Helmert | Lossy | 1–5 meters | RD New ↔ WGS84 via TOWGS84 |
| Grid-based (NTv2/GTX) | Lossy | 1–10 cm | RD New ↔ ETRS89 via RDNAPTRANS |
| Geographic ↔ Projected | Lossless | Exact (float precision only) | WGS84 geographic ↔ UTM zone 31N |
| Vertical datum shift | Lossy | Varies (cm to meters) | NAP ↔ ellipsoidal height |

**Key insight**: Transformations between different datums are ALWAYS lossy because they approximate the relationship between two different models of the Earth's shape. The accuracy depends on the method used and the geographic area covered.

---

## 2. IFC Georeferencing

IFC (Industry Foundation Classes) uses two entities to define how a BIM model's local coordinate system maps to a real-world CRS: **IfcProjectedCRS** and **IfcMapConversion**.

### IFC Hierarchy for Georeferencing

```
IfcProject
 └─ IfcGeometricRepresentationContext (type="Model")
     └─ HasCoordinateOperation → IfcMapConversion
                                   └─ TargetCRS → IfcProjectedCRS
```

The `IfcGeometricRepresentationContext` of the `IfcProject` serves as the `SourceCRS` for the `IfcMapConversion`. The `TargetCRS` MUST be an `IfcProjectedCRS` — geographic CRS definitions are NOT supported as targets.

### IfcProjectedCRS (IFC4 / IFC4X3)

Defines the target coordinate reference system. Inherits from `IfcCoordinateReferenceSystem`.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `Name` | IfcLabel | YES | CRS identifier, e.g., "EPSG:28992" |
| `Description` | IfcText | NO | Human-readable description |
| `GeodeticDatum` | IfcIdentifier | NO | Datum name, e.g., "Amersfoort" |
| `VerticalDatum` | IfcIdentifier | NO | Vertical datum name, e.g., "NAP" |
| `MapProjection` | IfcIdentifier | NO | Projection name, e.g., "Oblique Stereographic" |
| `MapZone` | IfcIdentifier | NO | Zone identifier, e.g., "31" for UTM zone 31 |
| `MapUnit` | IfcNamedUnit | NO | Unit of CRS axes (MUST be length unit) |

**Best practice**: ALWAYS populate the `Name` field with the full EPSG identifier (e.g., "EPSG:28992"). Many tools use ONLY this field to resolve the CRS definition. The other fields are informational and may be ignored by consuming software.

### IfcMapConversion (IFC4 / IFC4X3)

Defines the coordinate transformation from local BIM coordinates to the target CRS. Inherits from `IfcCoordinateOperation`.

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `Eastings` | IfcLengthMeasure | YES | — | Easting offset of local origin in target CRS |
| `Northings` | IfcLengthMeasure | YES | — | Northing offset of local origin in target CRS |
| `OrthogonalHeight` | IfcLengthMeasure | YES | — | Height of local origin relative to vertical datum |
| `XAxisAbscissa` | IfcReal | NO | 1.0 | Easting component of local X-axis direction vector |
| `XAxisOrdinate` | IfcReal | NO | 0.0 | Northing component of local X-axis direction vector |
| `Scale` | IfcReal | NO | 1.0 | Scale factor (for unit conversion between model and CRS) |

### IfcMapConversion Transformation Formula

The transformation from local coordinates (x, y, z) to map coordinates (E, N, H) is applied in three sequential steps:

**Step 1 — Scale** (uniform):
```
x' = x * Scale
y' = y * Scale
z' = z * Scale
```

**Step 2 — Rotate** (anti-clockwise about Z-axis):
```
θ = atan2(XAxisOrdinate, XAxisAbscissa)

E' = x' * cos(θ) - y' * sin(θ)
N' = x' * sin(θ) + y' * cos(θ)
H' = z'
```

**Step 3 — Translate**:
```
E = E' + Eastings
N = N' + Northings
H = H' + OrthogonalHeight
```

Or as a combined formula:
```
E = Scale * (x * XAxisAbscissa - y * XAxisOrdinate) + Eastings
N = Scale * (x * XAxisOrdinate + y * XAxisAbscissa) + Northings
H = Scale * z + OrthogonalHeight
```

Note: `XAxisAbscissa` and `XAxisOrdinate` represent a normalized direction vector: `cos(θ)` and `sin(θ)` respectively. The vector (XAxisAbscissa, XAxisOrdinate) MUST have unit length for a pure rotation. If the vector is not normalized, it introduces an additional scale factor.

### IFC4 vs IFC4X3 Differences

| Feature | IFC4 (ADD2 TC1) | IFC4X3 |
|---------|-----------------|--------|
| IfcMapConversion | Introduced | Retained, with formal proposition requiring TargetCRS to be IfcProjectedCRS |
| IfcMapConversionScaled | NOT available | NEW subtype — allows separate scale factors for X, Y, Z axes |
| IfcRigidOperation | NOT available | NEW — alternative for coordinate operations without scaling |
| IfcProjectedCRS | GeodeticDatum mandatory | GeodeticDatum changed to OPTIONAL |
| CRS Name convention | Free text | Convention to use "EPSG:XXXXX" format |

**IfcMapConversionScaled** (IFC4X3 only) adds `FactorX`, `FactorY`, and `FactorZ` attributes that multiply coordinates independently per axis — enabling non-uniform scaling. This is useful when the model uses different units per axis or when compensating for CRS scale distortions.

---

## 3. BIM Local Coordinate Systems

BIM authoring tools use local coordinate systems that bear no inherent relationship to real-world geography. Understanding how each tool defines its local system is critical for correct georeferencing.

### Revit Coordinate Concepts

- **Internal Origin**: Fixed, immutable point at (0,0,0). All geometry is stored relative to this.
- **Project Base Point**: User-movable reference for documentation. Does NOT affect geometry storage.
- **Survey Point**: Represents a known real-world point. Defines the relationship between model space and survey/site coordinates.
- **Project North vs True North**: Revit allows rotating True North relative to Project North. This rotation must be captured in `XAxisAbscissa`/`XAxisOrdinate` when exporting to IFC.
- **Internal units**: Feet (1 foot = 0.3048 meters exactly). All API calls return feet. Display units may differ.

### Blender / Bonsai Coordinate Concepts

- **World Origin**: (0,0,0) in Blender's coordinate system.
- **Axis convention**: Z-up, right-handed.
- **Default units**: Meters.
- **BIM integration**: Bonsai (BlenderBIM) reads/writes IFC. Georeferencing is handled via IfcMapConversion/IfcProjectedCRS entities. Bonsai provides UI panels for editing georeferencing parameters.
- **True North**: Stored as a 2D vector in `IfcGeometricRepresentationContext.TrueNorth` or derived from IfcMapConversion rotation.

### FreeCAD Coordinate Concepts

- **World Origin**: (0,0,0) in FreeCAD's coordinate system.
- **Axis convention**: Z-up, right-handed.
- **Default units**: Millimeters (internally).
- **BIM integration**: FreeCAD's Arch/BIM workbench can import/export IFC via IfcOpenShell. Georeferencing support is limited compared to Bonsai.

### Typical BIM Coordinate Ranges

| Unit System | Typical Range per Axis | Notes |
|-------------|----------------------|-------|
| Millimeters | 0 to 500,000 mm | Large buildings, ~500m span |
| Meters | 0 to 500 m | Same building in meters |
| Feet | 0 to 1,640 ft | Same building in feet |

**Warning**: When BIM models lack georeferencing, all geometry sits at or near the local origin (0,0,0). In map coordinates, this places the model at the intersection of the Equator and Prime Meridian (0°N, 0°E) — in the Gulf of Guinea, off the coast of West Africa. This is the single most common BIM↔GIS integration failure.

---

## 4. PROJ and pyproj — CRS Transformation Engine

### PROJ Overview

**PROJ** is the foundational open-source library for cartographic projections and coordinate transformations. It is used by virtually every open-source GIS tool (QGIS, GDAL, PostGIS, GeoPandas) and many commercial tools. PROJ supports:

- 200+ map projections
- Datum transformations (7-parameter Helmert, Molodensky, grid-based NTv2/GTX)
- 4D spatiotemporal coordinate operations
- Pipeline-based chained transformations

### pyproj — Python Interface to PROJ

**pyproj** provides Python bindings for PROJ. The two primary classes are `CRS` and `Transformer`.

#### Creating a CRS

```python
from pyproj import CRS

# From EPSG code
crs_rd = CRS.from_epsg(28992)

# From PROJ string
crs_utm31 = CRS.from_proj4("+proj=utm +zone=31 +datum=WGS84")

# From WKT
crs_wgs84 = CRS.from_epsg(4326)

# Inspect CRS properties
print(crs_rd.name)           # "Amersfoort / RD New"
print(crs_rd.is_projected)   # True
print(crs_rd.axis_info)      # [Axis(name='Easting', ...), Axis(name='Northing', ...)]
```

#### Coordinate Transformation

```python
from pyproj import Transformer

# Create transformer: WGS84 geographic → RD New
transformer = Transformer.from_crs(
    "EPSG:4326",   # source CRS
    "EPSG:28992",  # target CRS
    always_xy=True  # CRITICAL: forces (lon, lat) order, not (lat, lon)
)

# Transform a single point (Amsterdam Centraal)
easting, northing = transformer.transform(4.9003, 52.3792)
# Result: approximately (121687, 487316)

# Transform arrays (NumPy compatible)
import numpy as np
lons = np.array([4.9003, 5.1214, 4.3571])
lats = np.array([52.3792, 52.0907, 52.0116])
eastings, northings = transformer.transform(lons, lats)
```

**The `always_xy=True` flag is CRITICAL.** Without it, pyproj follows the CRS axis order definition. For EPSG:4326, the official axis order is (latitude, longitude), not (longitude, latitude). Setting `always_xy=True` forces consistent (x=lon, y=lat) ordering regardless of CRS definition. ALWAYS use this flag in AEC workflows.

#### Transformation Pipelines

```python
from pyproj import Transformer

# Custom pipeline: RD New → WGS84 via RDNAPTRANS (high accuracy)
pipeline = "+proj=pipeline " \
           "+step +inv +proj=sterea +lat_0=52.156160556 +lon_0=5.387638889 " \
           "+k=0.9999079 +x_0=155000 +y_0=463000 +ellps=bessel " \
           "+step +proj=hgridshift +grids=nl_nsgi_rdtrans2018.tif " \
           "+step +proj=unitconvert +xy_in=rad +xy_out=deg"

transformer = Transformer.from_pipeline(pipeline)
lon, lat = transformer.transform(155000, 463000)
```

#### Handling Vertical Datums

```python
from pyproj import Transformer

# 3D transformation: RD New + NAP → WGS84 geographic 3D
transformer_3d = Transformer.from_crs(
    "EPSG:7415",   # Amersfoort/RD New + NAP (compound CRS)
    "EPSG:4979",   # WGS84 geographic 3D (lat, lon, ellipsoidal height)
    always_xy=True
)

easting, northing, nap_height = 155000, 463000, 10.0
lon, lat, ellipsoidal_h = transformer_3d.transform(easting, northing, nap_height)
# ellipsoidal_h will differ from nap_height by the geoid separation (~42m in NL)
```

**Vertical datum warning**: NAP (Normaal Amsterdams Peil) heights differ from WGS84 ellipsoidal heights by the geoid undulation. In the Netherlands, this separation is approximately 42–44 meters. Confusing the two causes buildings to float 42 meters above the terrain or sink below it.

### PROJ Pipeline Syntax

PROJ pipelines chain operations sequentially. Each `+step` processes the output of the previous step:

```
+proj=pipeline
+step +proj=operation1 +param1=value1
+step +proj=operation2 +param2=value2
+step +inv +proj=operation3   # +inv reverses the operation
```

Key operations:
- `+proj=utm`: UTM projection
- `+proj=sterea`: Oblique Stereographic (used by RD New)
- `+proj=cart`: Geographic to geocentric Cartesian
- `+proj=helmert`: 7-parameter datum transformation
- `+proj=hgridshift`: Horizontal grid shift (NTv2)
- `+proj=vgridshift`: Vertical grid shift (GTX/GeoTIFF)
- `+proj=unitconvert`: Unit conversion (radians ↔ degrees, etc.)

---

## 5. QGIS↔BIM Integration

### Current State (2026)

QGIS does NOT natively import IFC files. The integration path requires intermediate conversion steps. There are several approaches:

### Approach 1: GDAL IFC Driver

GDAL includes an IFC driver (based on IfcOpenShell) that can read IFC files as vector layers. However, this driver has significant limitations:

- Geometry support is limited to 2D/2.5D footprints
- Complex BIM geometry (curved surfaces, voids) is often simplified or lost
- CRS information from IfcProjectedCRS/IfcMapConversion may not be automatically applied
- Performance degrades with large IFC files

### Approach 2: IfcOpenShell + PyQGIS (Recommended)

The most reliable approach uses IfcOpenShell to extract geometry and attributes from IFC, transform coordinates using the IfcMapConversion parameters, and create GIS layers in QGIS via PyQGIS.

```python
import ifcopenshell
import ifcopenshell.util.geolocation as geo
from qgis.core import (
    QgsProject, QgsVectorLayer, QgsFeature,
    QgsGeometry, QgsPointXY, QgsCoordinateReferenceSystem,
    QgsField, QgsFields
)
from PyQt5.QtCore import QVariant

# Step 1: Open IFC and extract georeferencing
ifc = ifcopenshell.open("building.ifc")

# Extract IfcMapConversion parameters
map_conversion = ifc.by_type("IfcMapConversion")
if map_conversion:
    mc = map_conversion[0]
    eastings = mc.Eastings
    northings = mc.Northings
    ortho_height = mc.OrthogonalHeight
    x_abscissa = mc.XAxisAbscissa if mc.XAxisAbscissa else 1.0
    x_ordinate = mc.XAxisOrdinate if mc.XAxisOrdinate else 0.0
    scale = mc.Scale if mc.Scale else 1.0

# Extract IfcProjectedCRS
projected_crs = ifc.by_type("IfcProjectedCRS")
if projected_crs:
    crs_name = projected_crs[0].Name  # e.g., "EPSG:28992"

# Step 2: Extract wall footprints and transform to map coordinates
walls = ifc.by_type("IfcWall")
features = []
for wall in walls:
    # Get local placement coordinates
    placement = wall.ObjectPlacement
    local_x, local_y, local_z = 0.0, 0.0, 0.0  # extract from placement matrix

    # Transform local → map using IfcMapConversion
    e, n, h = geo.xyz2enh(
        local_x, local_y, local_z,
        eastings=eastings, northings=northings,
        orthogonal_height=ortho_height,
        x_axis_abscissa=x_abscissa,
        x_axis_ordinate=x_ordinate,
        scale=scale
    )
    features.append((e, n, h, wall.GlobalId, wall.Name))

# Step 3: Create QGIS vector layer
layer = QgsVectorLayer("Point?crs=EPSG:28992", "BIM Walls", "memory")
provider = layer.dataProvider()
provider.addAttributes([
    QgsField("GlobalId", QVariant.String),
    QgsField("Name", QVariant.String),
    QgsField("Height", QVariant.Double)
])
layer.updateFields()

for e, n, h, gid, name in features:
    feat = QgsFeature()
    feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(e, n)))
    feat.setAttributes([gid, name or "", h])
    provider.addFeature(feat)

layer.updateExtents()
QgsProject.instance().addMapLayer(layer)
```

### Approach 3: GeoPackage Intermediate

Convert IFC to GeoPackage (GPKG) using IfcOpenShell + GeoPandas/Shapely, then load in QGIS:

```python
import ifcopenshell
import ifcopenshell.util.geolocation as geo
import geopandas as gpd
from shapely.geometry import Point

# Extract and transform as above, then:
gdf = gpd.GeoDataFrame({
    'GlobalId': [f[3] for f in features],
    'Name': [f[4] for f in features],
    'Height': [f[2] for f in features],
    'geometry': [Point(f[0], f[1]) for f in features]
}, crs="EPSG:28992")

gdf.to_file("building_walls.gpkg", driver="GPKG")
# Open in QGIS: Layer → Add Layer → Add Vector Layer → building_walls.gpkg
```

### CRS Assignment in QGIS/PyQGIS

```python
from qgis.core import QgsCoordinateReferenceSystem, QgsCoordinateTransform, QgsProject

# Define source CRS (from IFC)
source_crs = QgsCoordinateReferenceSystem("EPSG:28992")

# Define target CRS (for display)
target_crs = QgsCoordinateReferenceSystem("EPSG:4326")

# Create transformation
xform = QgsCoordinateTransform(source_crs, target_crs, QgsProject.instance())

# Transform a point
point = xform.transform(QgsPointXY(155000, 463000))
# Returns approximately (5.387, 52.156)
```

---

## 6. Common Coordinate Failures

### Failure 1: Y-Up vs Z-Up Axis Mismatch

BIM tools (IFC, Blender, Revit, FreeCAD, QGIS) use **Z-up** conventions. Web-based 3D renderers (Three.js, Babylon.js) use **Y-up** conventions. Failing to swap Y↔Z axes causes models to appear rotated 90° — lying on their side.

**Fix**: When transferring from BIM to web viewer, swap Y and Z:
```
Three.js_x = BIM_x
Three.js_y = BIM_z   (BIM height becomes Three.js vertical)
Three.js_z = -BIM_y  (BIM northing becomes negative Three.js depth)
```

Note the negation of Y — this preserves the right-handed coordinate system.

### Failure 2: Unit Mismatch

| Scenario | Effect | Scale Factor |
|----------|--------|-------------|
| IFC in mm, CRS in m | Building appears 1000x too large | 0.001 |
| IFC in m, CRS in m | Correct | 1.0 |
| Revit feet to metric CRS | Building appears ~0.3x too small | 0.3048 |
| IFC in m, Scale omitted | Usually correct (default 1.0) | 1.0 |
| IFC in mm, Scale omitted | Building 1000x too large on map | BUG — Scale should be 0.001 |

**Fix**: ALWAYS check both the IFC project units (`IfcProject` → `IfcUnitAssignment`) AND the `IfcMapConversion.Scale` value. The Scale factor should convert from project length units to CRS map units.

### Failure 3: Missing Georeferencing

When IfcMapConversion and IfcProjectedCRS are absent, the model has no geographic context. Importing it into GIS places it at the local origin, which maps to (0,0) in whatever CRS is assumed — often near (0°N, 0°E) in geographic coordinates.

**Detection**:
```python
import ifcopenshell

ifc = ifcopenshell.open("model.ifc")
has_georef = len(ifc.by_type("IfcMapConversion")) > 0
has_crs = len(ifc.by_type("IfcProjectedCRS")) > 0

if not has_georef or not has_crs:
    print("WARNING: Model lacks georeferencing. Cannot place on map.")
```

### Failure 4: Wrong CRS Assignment

Assigning the wrong EPSG code (e.g., using EPSG:32631 UTM zone 31N when the coordinates are actually in EPSG:28992 RD New) causes the model to appear in the wrong location, typically hundreds of kilometers off.

**Detection**: Check if the coordinate values are plausible for the assigned CRS:

| CRS | Valid Easting Range | Valid Northing Range |
|-----|-------------------|---------------------|
| EPSG:28992 (RD New) | -7,000 to 300,000 | 289,000 to 629,000 |
| EPSG:32631 (UTM 31N) | 166,000 to 834,000 | 0 to 9,400,000 |
| EPSG:32632 (UTM 32N) | 166,000 to 834,000 | 0 to 9,400,000 |

### Failure 5: True North Rotation Error

BIM models often have Project North aligned with the building grid (for ease of modeling), while True North is rotated. If the rotation angle stored in `XAxisAbscissa`/`XAxisOrdinate` is wrong, the building appears rotated on the map.

**Verification**: For a 30° anti-clockwise True North rotation:
```
XAxisAbscissa = cos(30°) = 0.866025
XAxisOrdinate = sin(30°) = 0.5
θ = atan2(0.5, 0.866025) = 30°
```

### Failure 6: Scale Factor Near UTM Zone Boundaries

UTM projections have a scale factor of 0.9996 at the central meridian, rising to 1.0004 at the zone edges. This means a 1-meter object at the zone edge is represented as 1.0004 meters in UTM coordinates — a 0.04% error. For a 100-meter building, this is a 4cm discrepancy. Near zone boundaries, coordinates from adjacent zones may both be valid but differ by hundreds of kilometers in easting.

**Fix**: NEVER mix coordinates from different UTM zones. Use a single CRS for the entire project area.

### Failure 7: Vertical Datum Confusion

Confusing NAP heights with WGS84 ellipsoidal heights introduces an error of approximately 42–44 meters in the Netherlands. This causes:
- Buildings floating above terrain in 3D viewers
- Incorrect elevation clash detection
- Wrong flood risk analysis

---

## 7. Axis Conventions by Tool — Comparison Table

| Tool | Up Axis | Handedness | Default Units | Coordinate Order | Notes |
|------|---------|-----------|---------------|-----------------|-------|
| **Blender** | Z-up | Right-handed | Meters | (X, Y, Z) | Bonsai aligns with IFC conventions |
| **Three.js** | Y-up | Right-handed | Unitless | (X, Y, Z) | Must swap Y↔Z from BIM data |
| **QGIS** | Z-up (2.5D) | Right-handed | Map units (m or deg) | (X=Easting, Y=Northing) | Z only for 2.5D features |
| **IFC** | Z-up | Right-handed | Project units (mm or m) | (X, Y, Z) | Units defined in IfcUnitAssignment |
| **Revit** | Z-up | Right-handed | Internal: feet | (X, Y, Z) | API returns feet, display may differ |
| **FreeCAD** | Z-up | Right-handed | Internal: mm | (X, Y, Z) | Arch workbench defaults to mm |
| **Speckle** | Z-up | Right-handed | Meters | (X, Y, Z) | Converts on receive per target app |
| **web-ifc** | Z-up | Right-handed | IFC project units | (X, Y, Z) | Same as source IFC file |
| **CityGML/3DTiles** | Z-up | Right-handed | Meters | (X=Easting, Y=Northing, Z) | Often in ECEF or local tangent plane |

### Critical Transformation: BIM (Z-up) → Three.js (Y-up)

```
// JavaScript: IFC Z-up → Three.js Y-up
function bimToThreeJs(x, y, z) {
    return {
        x: x,       // BIM X → Three.js X (east)
        y: z,        // BIM Z (up) → Three.js Y (up)
        z: -y        // BIM Y (north) → Three.js -Z (forward)
    };
}
```

This negation of Y→Z is necessary to preserve the right-handed coordinate system. Without the negation, the model appears mirrored.

---

## 8. IfcOpenShell Georeferencing API

IfcOpenShell (version 0.8.x) provides dedicated APIs for georeferencing operations.

### High-Level API: `ifcopenshell.api.georeference`

| Function | Purpose |
|----------|---------|
| `add_georeferencing(ifc_file, ifc_class, name)` | Creates empty IfcProjectedCRS + IfcMapConversion. Default class: "IfcMapConversion", default name: "EPSG:3857". |
| `edit_georeferencing(ifc_file, coordinate_operation, projected_crs)` | Updates IfcMapConversion and IfcProjectedCRS attributes. |
| `edit_true_north(ifc_file, true_north)` | Sets true north as a 2D vector or angle in degrees (positive = anti-clockwise). |
| `edit_wcs(ifc_file, x, y, z, rotation, is_si)` | Adjusts the World Coordinate System origin and rotation. |
| `remove_georeferencing(ifc_file)` | Removes all georeferencing entities. |

### Low-Level Utility: `ifcopenshell.util.geolocation`

| Function | Signature | Purpose |
|----------|-----------|---------|
| `xyz2enh` | `(x, y, z, eastings, northings, orthogonal_height, x_axis_abscissa, x_axis_ordinate, scale, factor_x, factor_y, factor_z) → (E, N, H)` | Local → Map coordinates |
| `enh2xyz` | `(e, n, h, ...) → (X, Y, Z)` | Map → Local coordinates |
| `local2global` | `(matrix, ...) → matrix` | 4x4 matrix from local to global |
| `global2local` | `(matrix, ...) → matrix` | 4x4 matrix from global to local |
| `auto_xyz2enh` | Auto-detected from IFC file | Automatic local → map (reads IfcMapConversion) |
| `auto_enh2xyz` | Auto-detected from IFC file | Automatic map → local |

The `factor_x`, `factor_y`, `factor_z` parameters correspond to the IFC4X3 `IfcMapConversionScaled` attributes. For IFC4 models, these default to 1.0.

### Complete Georeferencing Workflow with IfcOpenShell

```python
import ifcopenshell
import ifcopenshell.api

# Create a new IFC file with georeferencing
ifc = ifcopenshell.api.run("project.create_file", version="IFC4")
project = ifcopenshell.api.run("root.create_entity", ifc, ifc_class="IfcProject")

# Add georeferencing (creates IfcProjectedCRS + IfcMapConversion)
ifcopenshell.api.run("georeference.add_georeferencing", ifc)

# Set CRS to RD New
ifcopenshell.api.run("georeference.edit_georeferencing", ifc,
    projected_crs={"Name": "EPSG:28992"},
    coordinate_operation={
        "Eastings": 155000.0,
        "Northings": 463000.0,
        "OrthogonalHeight": 0.0,
        "XAxisAbscissa": 1.0,   # No rotation (project north = true north)
        "XAxisOrdinate": 0.0,
        "Scale": 1.0            # Model units = CRS units (meters)
    }
)

# Set true north (if different from project north)
# Example: true north is 15° anti-clockwise from project north
ifcopenshell.api.run("georeference.edit_true_north", ifc, true_north=15.0)

ifc.write("georeferenced_model.ifc")
```

---

## 9. End-to-End BIM↔GIS Pipeline

A complete pipeline for placing a BIM model on a map involves these steps:

```
IFC File
  │
  ├─ 1. Extract georeferencing (IfcMapConversion + IfcProjectedCRS)
  │     └─ If missing: STOP — model cannot be placed without manual intervention
  │
  ├─ 2. Extract geometry (IfcOpenShell geometric processing)
  │     └─ Convert to 2D footprints or 3D meshes
  │
  ├─ 3. Transform local → map CRS (xyz2enh using IfcMapConversion params)
  │     └─ Check: Scale, rotation, unit consistency
  │
  ├─ 4. Optional: Reproject to different CRS (pyproj Transformer)
  │     └─ Example: EPSG:28992 → EPSG:4326 for web maps
  │
  ├─ 5. Create GIS layer (GeoPackage, GeoJSON, Shapefile)
  │     └─ Assign correct CRS to output
  │
  └─ 6. Load in GIS tool (QGIS, web map, PostGIS)
        └─ Verify: building overlays known landmarks correctly
```

### Precision Considerations

| Operation | Precision Loss | Mitigation |
|-----------|---------------|-----------|
| Float64 coordinates | ~15 significant digits | Sufficient for all AEC work |
| mm → m conversion | None (exact division) | N/A |
| feet → m conversion | None (0.3048 is exact) | N/A |
| Datum transformation (Helmert) | 1–5 m | Use grid-based transforms |
| Datum transformation (grid) | 1–10 cm | Use latest grid files |
| Vertical datum conversion | 1–5 cm (with grid) | ALWAYS use geoid grids |
| CRS reprojection | Sub-mm at typical BIM scales | Negligible |

### Validation Checklist

1. Does the IFC file contain IfcMapConversion AND IfcProjectedCRS?
2. Is the EPSG code in IfcProjectedCRS.Name valid and correct for the project location?
3. Are Eastings/Northings values within the valid range for the assigned CRS?
4. Does the Scale factor correctly convert from project units to CRS units?
5. Is the True North rotation consistent between IfcMapConversion and IfcGeometricRepresentationContext?
6. When placed on a map, does the building overlay known landmarks (roads, parcels)?
7. Is the vertical datum (NAP vs ellipsoidal) consistently used throughout the pipeline?

---

## 10. Key Takeaways for Skill Development

1. **ALWAYS use `always_xy=True`** in pyproj to avoid axis order confusion.
2. **ALWAYS check for IfcMapConversion** before attempting BIM→GIS conversion. Without it, coordinates are meaningless in geographic context.
3. **NEVER assume units** — verify both IFC project units and IfcMapConversion.Scale.
4. **NEVER mix UTM zones** — use a single CRS for the entire project area.
5. **ALWAYS distinguish NAP height from ellipsoidal height** — the ~42m difference in NL causes visible errors.
6. **ALWAYS apply Y↔Z swap** when moving data between Z-up (BIM/GIS) and Y-up (Three.js) environments.
7. **ALWAYS validate** by overlaying on a known map after transformation — visual verification catches errors that math alone misses.
8. **NEVER hardcode TOWGS84 parameters** — use pyproj/PROJ with EPSG codes, which automatically selects the best available transformation for the area.
9. **For Dutch AEC projects**, prefer EPSG:7415 (compound: RD New + NAP) over bare EPSG:28992 to correctly handle vertical coordinates.
10. **IfcOpenShell's `auto_xyz2enh`** is the preferred function for coordinate conversion — it reads IfcMapConversion parameters directly from the model file.

---

## Sources

- buildingSMART IFC4 ADD2 TC1: https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/
- buildingSMART IFC4X3: https://ifc43-docs.standards.buildingsmart.org/
- PROJ documentation: https://proj.org/en/stable/
- pyproj documentation: https://pyproj4.github.io/pyproj/stable/
- IfcOpenShell documentation: https://docs.ifcopenshell.org/
- EPSG registry: https://epsg.org / https://epsg.io
- QGIS documentation: https://docs.qgis.org
- GDAL IFC driver: https://gdal.org/en/stable/drivers/vector/ifc.html
