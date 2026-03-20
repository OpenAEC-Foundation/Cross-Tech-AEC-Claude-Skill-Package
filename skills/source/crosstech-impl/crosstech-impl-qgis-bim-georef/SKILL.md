---
name: crosstech-impl-qgis-bim-georef
description: >
  Use when georeferencing BIM models in QGIS, importing IFC footprints onto maps,
  or transforming coordinates between BIM local and GIS global systems.
  Prevents the common mistake of importing IFC without CRS assignment or
  losing the third dimension during 2D projection.
  Covers IfcOpenShell+PyQGIS extraction, CRS transformation via IfcMapConversion,
  GeoPackage export, and three integration approaches (GDAL, IfcOpenShell, Speckle).
  Keywords: QGIS, BIM, georeferencing, IFC, GIS, CRS, IfcMapConversion, PyQGIS,
  footprint extraction, GeoPackage, coordinate transformation, map overlay.
license: MIT
compatibility: "Designed for Claude Code. Covers QGIS 3.34 LTR, IfcOpenShell 0.8.x, pyproj 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-impl-qgis-bim-georef

## Quick Reference

### Integration Approaches

| Approach | Flexibility | Geometry | Effort | Status |
|----------|------------|----------|--------|--------|
| GDAL/OGR | Low | None | Low | NO IFC driver (GDAL 3.9) |
| IfcOpenShell + PyQGIS | High | Full 3D | High | Recommended |
| Speckle QGIS Connector | Medium | 2D + basic 3D | Medium | Legacy connector |

### Critical Warnings

**NEVER** use GDAL/OGR to read IFC files directly into QGIS — GDAL does NOT have a native IFC driver as of GDAL 3.9. The URL `https://gdal.org/en/stable/drivers/vector/ifc.html` returns 404.

**NEVER** import IFC geometry into QGIS without first checking for `IfcMapConversion` — models without georeferencing land at (0,0) in whatever CRS is assumed.

**NEVER** use Shapefile format for BIM-to-GIS export — Shapefiles drop Z values and limit field names to 10 characters. ALWAYS use GeoPackage.

**NEVER** skip floor-by-floor filtering before 2D projection — multi-storey elements overlap in the same footprint.

**ALWAYS** use `settings.set(settings.USE_WORLD_COORDS, True)` when extracting geometry with IfcOpenShell — otherwise geometry stays in local element placement.

**ALWAYS** verify unit consistency between IFC project units and CRS map units before coordinate transformation.

**ALWAYS** use the IfcOpenShell geometry iterator for batch processing — `create_shape()` per element is significantly slower.

---

## Technology Boundary

### Side A: QGIS (GIS World)

QGIS is a 2D map-centric GIS with limited 3D support (QGIS 3D viewer). Layers carry CRS metadata and all coordinates are georeferenced.

**Key concepts:**
- **Layers**: Vector (points, lines, polygons) or raster, each with an assigned CRS
- **CRS handling**: Automatic on-the-fly reprojection between layers with different CRS
- **PyQGIS API**: `qgis.core` module provides `QgsVectorLayer`, `QgsFeature`, `QgsGeometry`, `QgsCoordinateReferenceSystem`, `QgsCoordinateTransform`
- **Supported formats**: GeoPackage (recommended), Shapefile, GeoJSON, PostGIS, WMS/WFS
- **Coordinate order**: ALWAYS (easting, northing) / (x, y) in PyQGIS API

**QGIS 3.34 LTR specifics:**
- Python 3.x bindings via PyQt5
- Native GeoPackage read/write with spatial indexing
- 3D viewer supports 2.5D extrusion and mesh layers

### Side B: BIM/IFC (Building World)

BIM models use local coordinate systems with no inherent geographic meaning. Geometry is defined relative to a local origin.

**Key concepts:**
- **Local coordinates**: All geometry positioned relative to project origin (0,0,0)
- **IfcOpenShell 0.8.x**: Python library for IFC parsing, geometry extraction, and coordinate transformation
- **Geometry iterator**: `ifcopenshell.geom.iterator()` for efficient batch geometry extraction
- **Georeferencing entities**: `IfcMapConversion` (transformation parameters) + `IfcProjectedCRS` (target CRS)
- **Project units**: Defined in `IfcProject` -> `IfcUnitAssignment` (typically mm or m)

**IfcOpenShell georeferencing API:**
- `ifcopenshell.util.geolocation.auto_xyz2enh()` — local to map coordinates (reads parameters from file)
- `ifcopenshell.util.geolocation.xyz2enh()` — local to map with explicit parameters
- `ifcopenshell.util.geolocation.enh2xyz()` — map to local (reverse direction)

### The Bridge: IfcOpenShell Extraction + pyproj CRS Transformation

The bridge operates in two stages:

**Stage 1 — IFC Local to Projected CRS** (via IfcMapConversion):
```
E = Scale * (x * cos(theta) - y * sin(theta)) + Eastings
N = Scale * (x * sin(theta) + y * cos(theta)) + Northings
H = Scale * z + OrthogonalHeight
```

**Stage 2 — Projected CRS to Any Target CRS** (via pyproj):
```python
from pyproj import Transformer
transformer = Transformer.from_crs("EPSG:28992", "EPSG:4326", always_xy=True)
lon, lat = transformer.transform(easting, northing)
```

**Stage 3 — Load into QGIS** (via PyQGIS or GeoPackage):
```
IFC File -> IfcOpenShell (extract + transform) -> GeoDataFrame -> GeoPackage -> QGIS Layer
```

**Directionality**: Primarily IFC -> QGIS. Limited QGIS -> IFC is possible via `enh2xyz()` but requires an existing georeferenced IFC file as target.

> For CRS fundamentals, EPSG codes, and axis conventions, see `crosstech-core-coordinate-systems`.

---

## Critical Rules (ALWAYS/NEVER)

1. **ALWAYS** check for both `IfcMapConversion` AND `IfcProjectedCRS` before attempting BIM-to-GIS conversion.
2. **ALWAYS** set `always_xy=True` in pyproj `Transformer.from_crs()` calls — EPSG:4326 defaults to (lat, lon) order without it.
3. **ALWAYS** use GeoPackage format for BIM-GIS export — it preserves Z values, supports long field names, and includes spatial indexing.
4. **ALWAYS** filter by building storey before projecting 3D BIM to 2D footprints — prevents overlapping polygons.
5. **ALWAYS** use `ifcopenshell.geom.iterator()` with multiprocessing for batch geometry extraction.
6. **ALWAYS** verify extracted coordinates fall within the valid range for the assigned CRS.
7. **NEVER** assume IFC project units are meters — read from `IfcProject` -> `IfcUnitAssignment`.
8. **NEVER** use EPSG:3857 (Web Mercator) for engineering coordinate work — it distorts areas and distances.
9. **NEVER** mix vertical datums (NAP vs ellipsoidal) without explicit conversion via compound CRS.
10. **NEVER** rely on the Speckle QGIS connector for production BIM-GIS workflows — it is a legacy connector with limited geometry support.

---

## Decision Tree

```
Need to get BIM data into QGIS?
|
+-- Does the IFC file contain IfcMapConversion + IfcProjectedCRS?
|   |
|   +-- YES -> Use IfcOpenShell + PyQGIS approach (Pattern 1-4)
|   |   |
|   |   +-- Need 2D footprints? -> Extract with Shapely convex hull (Pattern 3)
|   |   +-- Need point locations? -> Extract centroids (Pattern 2)
|   |   +-- Need polygon outlines? -> Use geometry iterator (Pattern 4)
|   |   |
|   |   +-- Target CRS differs from IFC CRS?
|   |       +-- YES -> Chain with pyproj Transformer (Pattern 5)
|   |       +-- NO -> Use coordinates directly
|   |
|   +-- NO -> STOP. Add georeferencing first:
|       +-- ifcopenshell.api.run("georeference.add_georeferencing")
|       +-- See crosstech-core-coordinate-systems Pattern 5
|
+-- Need automated pipeline (Revit -> QGIS)?
|   +-- Use Speckle as transport layer (limited geometry)
|   +-- Or export IFC from Revit, then use IfcOpenShell approach
|
+-- Model appears at wrong location in QGIS?
    +-- At origin / Gulf of Guinea? -> Missing IfcMapConversion
    +-- Hundreds of km off? -> Wrong EPSG code
    +-- 1000x too large? -> Unit mismatch (mm vs m)
    +-- Rotated? -> Wrong XAxisAbscissa/XAxisOrdinate
    +-- ~42m vertical offset (NL)? -> NAP vs ellipsoidal height
```

---

## Essential Patterns

### Pattern 1: Check Georeferencing and Extract CRS

```python
import ifcopenshell

ifc = ifcopenshell.open("building.ifc")

# Check for georeferencing entities
map_conversions = ifc.by_type("IfcMapConversion")
projected_crs_list = ifc.by_type("IfcProjectedCRS")

if not map_conversions or not projected_crs_list:
    raise ValueError("IFC file lacks georeferencing — cannot place on map")

mc = map_conversions[0]
crs = projected_crs_list[0]

# Extract transformation parameters
eastings = mc.Eastings
northings = mc.Northings
ortho_height = mc.OrthogonalHeight
x_abscissa = mc.XAxisAbscissa if mc.XAxisAbscissa else 1.0
x_ordinate = mc.XAxisOrdinate if mc.XAxisOrdinate else 0.0
scale = mc.Scale if mc.Scale else 1.0

# Extract CRS identifier
epsg_code = crs.Name  # e.g., "EPSG:28992"
```

### Pattern 2: IFC Elements to QGIS Point Layer

```python
import ifcopenshell
import ifcopenshell.util.geolocation as geo
from qgis.core import (
    QgsVectorLayer, QgsProject, QgsFeature,
    QgsGeometry, QgsPointXY, QgsField
)
from PyQt5.QtCore import QVariant

ifc = ifcopenshell.open("building.ifc")

# Create memory layer with CRS from IFC
layer = QgsVectorLayer("Point?crs=EPSG:28992", "BIM Elements", "memory")
provider = layer.dataProvider()
provider.addAttributes([
    QgsField("GlobalId", QVariant.String),
    QgsField("Name", QVariant.String),
    QgsField("IfcType", QVariant.String),
    QgsField("Height", QVariant.Double),
])
layer.updateFields()

for wall in ifc.by_type("IfcWall"):
    # auto_xyz2enh reads IfcMapConversion from the file
    local_x, local_y, local_z = 0.0, 0.0, 0.0  # Extract from ObjectPlacement
    e, n, h = geo.auto_xyz2enh(ifc, local_x, local_y, local_z)

    feat = QgsFeature()
    feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(e, n)))
    feat.setAttributes([wall.GlobalId, wall.Name or "", wall.is_a(), h])
    provider.addFeature(feat)

layer.updateExtents()
QgsProject.instance().addMapLayer(layer)
```

### Pattern 3: IFC Footprint Extraction to GeoPackage

```python
import ifcopenshell
import ifcopenshell.geom
import ifcopenshell.util.geolocation as geo
import multiprocessing
from shapely.geometry import MultiPoint
import geopandas as gpd

ifc = ifcopenshell.open("building.ifc")

# Configure geometry extraction with world coordinates
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)
settings.set(settings.WELD_VERTICES, True)

# Use iterator for efficient batch processing
iterator = ifcopenshell.geom.iterator(
    settings, ifc, multiprocessing.cpu_count()
)

features = []
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = ifc.by_id(shape.id)
        verts = shape.geometry.verts

        # Extract XY coordinates (drop Z for 2D footprint)
        coords_2d = [(verts[i], verts[i + 1])
                      for i in range(0, len(verts), 3)]

        if len(coords_2d) >= 3:
            footprint = MultiPoint(coords_2d).convex_hull

            # Transform local coords to map coords
            centroid = footprint.centroid
            e, n, h = geo.auto_xyz2enh(
                ifc, centroid.x, centroid.y, 0.0
            )

            features.append({
                "geometry": footprint,
                "GlobalId": element.GlobalId,
                "IfcType": element.is_a(),
                "Name": getattr(element, "Name", None) or "",
            })

        if not iterator.next():
            break

# Export to GeoPackage with CRS
gdf = gpd.GeoDataFrame(features, crs="EPSG:28992")
gdf.to_file("building_footprints.gpkg", driver="GPKG")
```

### Pattern 4: Load GeoPackage into QGIS

```python
from qgis.core import (
    QgsVectorLayer, QgsProject, QgsCoordinateReferenceSystem
)

layer = QgsVectorLayer(
    "building_footprints.gpkg", "BIM Footprints", "ogr"
)

if not layer.isValid():
    raise RuntimeError("Failed to load GeoPackage layer")

# Verify CRS is set (GeoPackage embeds CRS, but verify)
if not layer.crs().isValid():
    layer.setCrs(QgsCoordinateReferenceSystem("EPSG:28992"))

QgsProject.instance().addMapLayer(layer)
```

### Pattern 5: CRS Reprojection in QGIS

```python
from qgis.core import (
    QgsCoordinateReferenceSystem, QgsCoordinateTransform,
    QgsProject, QgsPointXY
)

source_crs = QgsCoordinateReferenceSystem("EPSG:28992")
target_crs = QgsCoordinateReferenceSystem("EPSG:4326")

xform = QgsCoordinateTransform(
    source_crs, target_crs, QgsProject.instance()
)

# Transform a point from RD New to WGS84
point_wgs84 = xform.transform(QgsPointXY(155000, 463000))
# Returns approximately (5.387, 52.156)
```

---

## Common Operations

### Overlay BIM Footprints on OpenStreetMap

```python
from qgis.core import QgsRasterLayer, QgsProject

# Add OpenStreetMap basemap via XYZ tiles
osm_url = (
    "type=xyz&url=https://tile.openstreetmap.org/"
    "{z}/{x}/{y}.png&zmax=19&zmin=0"
)
basemap = QgsRasterLayer(osm_url, "OpenStreetMap", "wms")

if basemap.isValid():
    QgsProject.instance().addMapLayer(basemap, False)
    # Insert below vector layers
    root = QgsProject.instance().layerTreeRoot()
    root.insertLayer(-1, basemap)
```

### Filter by Building Storey Before Projection

```python
import ifcopenshell

ifc = ifcopenshell.open("building.ifc")

# Get elements on a specific storey
target_storey = "Ground Floor"
for storey in ifc.by_type("IfcBuildingStorey"):
    if storey.Name == target_storey:
        # Get all elements decomposed into this storey
        for rel in storey.IsDecomposedBy:
            for element in rel.RelatedObjects:
                # Process only elements on this storey
                pass
        # Get elements contained in this storey
        for rel in storey.ContainsElements:
            for element in rel.RelatedElements:
                # Process only elements on this storey
                pass
```

### Validate Coordinates Against CRS Range

```python
CRS_RANGES = {
    "EPSG:28992": {"e_min": -7000, "e_max": 300000,
                    "n_min": 289000, "n_max": 629000},
    "EPSG:32631": {"e_min": 166000, "e_max": 834000,
                    "n_min": 0, "n_max": 9400000},
    "EPSG:32632": {"e_min": 166000, "e_max": 834000,
                    "n_min": 0, "n_max": 9400000},
}

def validate_coordinates(easting, northing, epsg):
    bounds = CRS_RANGES.get(epsg)
    if not bounds:
        return True  # Unknown CRS, cannot validate
    if not (bounds["e_min"] <= easting <= bounds["e_max"]):
        raise ValueError(f"Easting {easting} outside valid range for {epsg}")
    if not (bounds["n_min"] <= northing <= bounds["n_max"]):
        raise ValueError(f"Northing {northing} outside valid range for {epsg}")
    return True
```

---

## Data Loss at This Boundary

| IFC Data | Preserved in QGIS | Lost in QGIS |
|----------|-------------------|---------------|
| 2D geometry (footprints) | YES (polygons) | |
| Z-coordinates | YES (GeoPackage) | YES (Shapefile) |
| GlobalId | YES (attribute) | |
| Element name/type | YES (attributes) | |
| Property sets | Partially (flat columns) | Nested hierarchy |
| Material associations | NO | YES |
| Parametric definitions | NO | YES |
| Boolean operations | Resolved by IfcOpenShell | Original CSG tree |
| Curved geometry | Tessellated to polygons | Exact curves |
| Spatial hierarchy | Partially (storey attribute) | IfcRelAggregates tree |

---

## Reference Links

- [references/methods.md](references/methods.md) — PyQGIS and IfcOpenShell API signatures for georeferencing
- [references/examples.md](references/examples.md) — Complete working pipelines for IFC-to-QGIS workflows
- [references/anti-patterns.md](references/anti-patterns.md) — Common BIM-GIS integration failures and how to avoid them

### Official Sources

- https://docs.qgis.org/3.34/en/docs/pyqgis_developer_cookbook/ (PyQGIS Developer Cookbook)
- https://docs.ifcopenshell.org/ (IfcOpenShell documentation)
- https://pyproj4.github.io/pyproj/stable/ (pyproj documentation)
- https://gdal.org/en/stable/drivers/vector/index.html (GDAL vector drivers — NO IFC driver)
- https://geopandas.org/en/stable/ (GeoPandas documentation)
- https://shapely.readthedocs.io/en/stable/ (Shapely documentation)

### Related Skills

- `crosstech-core-coordinate-systems` — CRS fundamentals, EPSG codes, IfcMapConversion math
- `crosstech-impl-speckle-qgis` — Speckle-based BIM-to-QGIS transport (alternative approach)
