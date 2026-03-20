# crosstech-impl-qgis-bim-georef — Methods Reference

## IfcOpenShell Georeferencing API

### ifcopenshell.util.geolocation.auto_xyz2enh()

Converts local BIM coordinates to map coordinates by reading IfcMapConversion from the file.

```python
import ifcopenshell.util.geolocation as geo

e, n, h = geo.auto_xyz2enh(ifc_file, x, y, z)
```

**Parameters:**
- `ifc_file` (ifcopenshell.file): Open IFC file with IfcMapConversion
- `x`, `y`, `z` (float): Local BIM coordinates

**Returns:** Tuple `(easting, northing, height)` in the CRS defined by IfcProjectedCRS

### ifcopenshell.util.geolocation.xyz2enh()

Converts local BIM coordinates to map coordinates with explicit parameters.

```python
e, n, h = geo.xyz2enh(
    x, y, z,
    eastings=155000.0,
    northings=463000.0,
    orthogonal_height=0.0,
    x_axis_abscissa=1.0,
    x_axis_ordinate=0.0,
    scale=1.0
)
```

**Parameters:**
- `x`, `y`, `z` (float): Local BIM coordinates
- `eastings` (float): IfcMapConversion.Eastings
- `northings` (float): IfcMapConversion.Northings
- `orthogonal_height` (float): IfcMapConversion.OrthogonalHeight
- `x_axis_abscissa` (float): cos(true north rotation)
- `x_axis_ordinate` (float): sin(true north rotation)
- `scale` (float): Conversion factor from IFC units to CRS units

### ifcopenshell.util.geolocation.enh2xyz()

Reverse transformation: map coordinates to local BIM coordinates.

```python
x, y, z = geo.enh2xyz(
    e, n, h,
    eastings=155000.0,
    northings=463000.0,
    orthogonal_height=0.0,
    x_axis_abscissa=1.0,
    x_axis_ordinate=0.0,
    scale=1.0
)
```

---

## IfcOpenShell Geometry Extraction API

### ifcopenshell.geom.settings()

Configuration object for geometry extraction.

```python
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)   # Apply placement transforms
settings.set(settings.WELD_VERTICES, True)       # Merge duplicate vertices
settings.set(settings.APPLY_DEFAULT_MATERIALS, False)  # Skip materials
```

**Key settings:**
| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `USE_WORLD_COORDS` | bool | False | Apply ObjectPlacement transforms |
| `WELD_VERTICES` | bool | False | Merge coincident vertices |
| `APPLY_DEFAULT_MATERIALS` | bool | True | Include material data |

### ifcopenshell.geom.iterator()

Batch geometry processor — ALWAYS prefer over `create_shape()` for multiple elements.

```python
iterator = ifcopenshell.geom.iterator(settings, ifc_file, num_threads)
```

**Parameters:**
- `settings` (ifcopenshell.geom.settings): Geometry configuration
- `ifc_file` (ifcopenshell.file): Open IFC file
- `num_threads` (int): Number of parallel processing threads

**Usage:**
```python
if iterator.initialize():
    while True:
        shape = iterator.get()
        verts = shape.geometry.verts   # Flat: [x1,y1,z1, x2,y2,z2, ...]
        faces = shape.geometry.faces   # Flat: [i1,i2,i3, ...]
        element_id = shape.id          # IFC entity instance ID
        if not iterator.next():
            break
```

### ifcopenshell.geom.create_shape()

Single-element geometry extraction. Use ONLY when processing individual elements.

```python
shape = ifcopenshell.geom.create_shape(settings, element)
verts = shape.geometry.verts
faces = shape.geometry.faces
```

---

## PyQGIS Layer Creation API

### QgsVectorLayer (memory provider)

Creates an in-memory vector layer.

```python
from qgis.core import QgsVectorLayer

# Geometry types: "Point", "LineString", "Polygon", "MultiPoint",
#                 "MultiLineString", "MultiPolygon"
layer = QgsVectorLayer("Polygon?crs=EPSG:28992", "Layer Name", "memory")
```

### QgsVectorLayer.dataProvider()

Access the data provider for adding fields and features.

```python
from qgis.core import QgsField, QgsFeature, QgsGeometry, QgsPointXY
from PyQt5.QtCore import QVariant

provider = layer.dataProvider()

# Add fields
provider.addAttributes([
    QgsField("name", QVariant.String),
    QgsField("area", QVariant.Double),
    QgsField("storey", QVariant.Int),
])
layer.updateFields()

# Add features
feat = QgsFeature()
feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(155000, 463000)))
feat.setAttributes(["Wall-01", 25.5, 0])
provider.addFeature(feat)

layer.updateExtents()
```

### QgsCoordinateTransform

Transforms coordinates between CRS within QGIS.

```python
from qgis.core import (
    QgsCoordinateReferenceSystem, QgsCoordinateTransform, QgsProject
)

source = QgsCoordinateReferenceSystem("EPSG:28992")
target = QgsCoordinateReferenceSystem("EPSG:4326")
xform = QgsCoordinateTransform(source, target, QgsProject.instance())

point_transformed = xform.transform(QgsPointXY(155000, 463000))
```

---

## pyproj Transformation API

### Transformer.from_crs()

Creates a coordinate transformer between two CRS.

```python
from pyproj import Transformer

transformer = Transformer.from_crs(
    "EPSG:28992",      # Source CRS
    "EPSG:4326",       # Target CRS
    always_xy=True     # ALWAYS set this — prevents lat/lon order confusion
)

lon, lat = transformer.transform(155000, 463000)
```

**CRITICAL**: ALWAYS pass `always_xy=True`. Without it, EPSG:4326 uses (lat, lon) order, causing silent coordinate swaps.

---

## GeoPandas / Shapely API

### GeoDataFrame Creation and Export

```python
import geopandas as gpd
from shapely.geometry import Polygon, MultiPoint

# Create GeoDataFrame with CRS
gdf = gpd.GeoDataFrame(
    {"GlobalId": ["abc123"], "geometry": [Polygon([(0,0),(1,0),(1,1),(0,1)])]},
    crs="EPSG:28992"
)

# Export to GeoPackage
gdf.to_file("output.gpkg", driver="GPKG")

# Export to GeoJSON
gdf.to_file("output.geojson", driver="GeoJSON")
```

### Footprint from 3D Vertices

```python
from shapely.geometry import MultiPoint

# Drop Z, create convex hull
coords_2d = [(x, y) for x, y, z in vertices_3d]
footprint = MultiPoint(coords_2d).convex_hull
```

**Note**: Convex hull is an approximation. For concave footprints (L-shaped buildings), use `shapely.ops.unary_union` on triangulated faces or `alphashape` library.
