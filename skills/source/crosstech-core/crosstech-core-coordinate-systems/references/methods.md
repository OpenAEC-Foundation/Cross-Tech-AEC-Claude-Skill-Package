# methods.md — CRS Transformation APIs

## pyproj 3.x API

### CRS Class

```python
from pyproj import CRS

# From EPSG code
crs_rd = CRS.from_epsg(28992)

# From PROJ string
crs_utm31 = CRS.from_proj4("+proj=utm +zone=31 +datum=WGS84")

# From WKT string
crs_custom = CRS.from_wkt(wkt_string)

# Inspect properties
crs_rd.name           # "Amersfoort / RD New"
crs_rd.is_projected   # True
crs_rd.is_geographic  # False
crs_rd.axis_info      # [Axis(name='Easting', ...), Axis(name='Northing', ...)]
crs_rd.to_epsg()      # 28992
```

### Transformer Class

```python
from pyproj import Transformer

# Standard: CRS to CRS
transformer = Transformer.from_crs(
    "EPSG:4326",    # source CRS
    "EPSG:28992",   # target CRS
    always_xy=True  # ALWAYS set this — forces (x, y) = (lon, lat) order
)

# Single point
easting, northing = transformer.transform(4.9003, 52.3792)

# NumPy arrays (batch transformation)
import numpy as np
lons = np.array([4.9003, 5.1214, 4.3571])
lats = np.array([52.3792, 52.0907, 52.0116])
eastings, northings = transformer.transform(lons, lats)

# 3D transformation (with height)
transformer_3d = Transformer.from_crs(
    "EPSG:7415",   # RD New + NAP (compound)
    "EPSG:4979",   # WGS84 geographic 3D
    always_xy=True
)
lon, lat, ellipsoidal_h = transformer_3d.transform(155000, 463000, 10.0)

# Inverse transformation (swap source/target)
transformer_inv = Transformer.from_crs("EPSG:28992", "EPSG:4326", always_xy=True)
lon, lat = transformer_inv.transform(155000, 463000)
```

### Custom PROJ Pipelines

```python
from pyproj import Transformer

# High-accuracy RD New → WGS84 via RDNAPTRANS grid
pipeline = (
    "+proj=pipeline "
    "+step +inv +proj=sterea +lat_0=52.156160556 +lon_0=5.387638889 "
    "+k=0.9999079 +x_0=155000 +y_0=463000 +ellps=bessel "
    "+step +proj=hgridshift +grids=nl_nsgi_rdtrans2018.tif "
    "+step +proj=unitconvert +xy_in=rad +xy_out=deg"
)

transformer = Transformer.from_pipeline(pipeline)
lon, lat = transformer.transform(155000, 463000)
```

### PROJ Pipeline Syntax Reference

Each `+step` processes the output of the previous step:

| Operation | Syntax | Purpose |
|-----------|--------|---------|
| UTM projection | `+proj=utm +zone=31` | UTM forward/inverse |
| Oblique Stereographic | `+proj=sterea` | RD New projection |
| Geocentric Cartesian | `+proj=cart` | Geographic → XYZ |
| Helmert transform | `+proj=helmert +x=... +y=... +z=...` | 7-parameter datum shift |
| Horizontal grid shift | `+proj=hgridshift +grids=file.tif` | NTv2 grid transformation |
| Vertical grid shift | `+proj=vgridshift +grids=file.tif` | Geoid model application |
| Unit conversion | `+proj=unitconvert +xy_in=rad +xy_out=deg` | Radians ↔ degrees |
| Inverse step | `+step +inv +proj=...` | Reverse any operation |

---

## IfcOpenShell 0.8.x Georeferencing API

### High-Level API: `ifcopenshell.api.georeference`

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `add_georeferencing(ifc_file)` | Creates IfcProjectedCRS + IfcMapConversion | `ifc_class` (default: "IfcMapConversion"), `name` (default: "EPSG:3857") |
| `edit_georeferencing(ifc_file, ...)` | Updates georeferencing attributes | `projected_crs` (dict), `coordinate_operation` (dict) |
| `edit_true_north(ifc_file, true_north)` | Sets true north angle/vector | `true_north` (float degrees or 2D vector) |
| `edit_wcs(ifc_file, x, y, z, rotation)` | Adjusts World Coordinate System | `x`, `y`, `z`, `rotation`, `is_si` |
| `remove_georeferencing(ifc_file)` | Removes all georeferencing | — |

### Low-Level Utility: `ifcopenshell.util.geolocation`

| Function | Direction | Parameters |
|----------|-----------|------------|
| `xyz2enh(x, y, z, ...)` | Local → Map | `eastings`, `northings`, `orthogonal_height`, `x_axis_abscissa`, `x_axis_ordinate`, `scale` |
| `enh2xyz(e, n, h, ...)` | Map → Local | Same as above (inverse) |
| `auto_xyz2enh(ifc_file, x, y, z)` | Local → Map (auto) | Reads parameters from IfcMapConversion in the file |
| `auto_enh2xyz(ifc_file, e, n, h)` | Map → Local (auto) | Reads parameters from IfcMapConversion in the file |
| `local2global(matrix, ...)` | Local → Global | 4x4 transformation matrix |
| `global2local(matrix, ...)` | Global → Local | 4x4 transformation matrix |

IFC4X3 additional parameters for `xyz2enh` and `enh2xyz`: `factor_x`, `factor_y`, `factor_z` (for IfcMapConversionScaled). These default to 1.0 for IFC4.

### IfcMapConversion Attributes

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `Eastings` | IfcLengthMeasure | YES | — | Easting offset of local origin |
| `Northings` | IfcLengthMeasure | YES | — | Northing offset of local origin |
| `OrthogonalHeight` | IfcLengthMeasure | YES | — | Height relative to vertical datum |
| `XAxisAbscissa` | IfcReal | NO | 1.0 | cos(θ) of true north rotation |
| `XAxisOrdinate` | IfcReal | NO | 0.0 | sin(θ) of true north rotation |
| `Scale` | IfcReal | NO | 1.0 | Unit conversion factor |

### IfcProjectedCRS Attributes

| Attribute | Type | Required (IFC4) | Required (IFC4X3) | Description |
|-----------|------|-----------------|--------------------|----|
| `Name` | IfcLabel | YES | YES | CRS identifier, e.g., "EPSG:28992" |
| `Description` | IfcText | NO | NO | Human-readable description |
| `GeodeticDatum` | IfcIdentifier | YES | NO | Datum name |
| `VerticalDatum` | IfcIdentifier | NO | NO | Vertical datum name |
| `MapProjection` | IfcIdentifier | NO | NO | Projection name |
| `MapZone` | IfcIdentifier | NO | NO | Zone identifier |
| `MapUnit` | IfcNamedUnit | NO | NO | CRS axis unit |

---

## IfcMapConversion Transformation Formula

### Forward (Local → Map)

Step 1 — Scale:
```
x' = x * Scale
y' = y * Scale
z' = z * Scale
```

Step 2 — Rotate (anti-clockwise about Z-axis):
```
θ = atan2(XAxisOrdinate, XAxisAbscissa)
E' = x' * cos(θ) - y' * sin(θ)
N' = x' * sin(θ) + y' * cos(θ)
H' = z'
```

Step 3 — Translate:
```
E = E' + Eastings
N = N' + Northings
H = H' + OrthogonalHeight
```

Combined:
```
E = Scale * (x * XAxisAbscissa - y * XAxisOrdinate) + Eastings
N = Scale * (x * XAxisOrdinate + y * XAxisAbscissa) + Northings
H = Scale * z + OrthogonalHeight
```

### Inverse (Map → Local)

```
x' = (E - Eastings) * XAxisAbscissa + (N - Northings) * XAxisOrdinate
y' = (N - Northings) * XAxisAbscissa - (E - Eastings) * XAxisOrdinate
z' = H - OrthogonalHeight

x = x' / Scale
y = y' / Scale
z = z' / Scale
```

### IFC4X3 IfcMapConversionScaled Extension

For non-uniform scaling (IFC4X3 only):
```
E = FactorX * (x * XAxisAbscissa - y * XAxisOrdinate) + Eastings
N = FactorY * (x * XAxisOrdinate + y * XAxisAbscissa) + Northings
H = FactorZ * z + OrthogonalHeight
```

Where `FactorX`, `FactorY`, `FactorZ` replace the uniform `Scale` parameter.

---

## QGIS CRS API (PyQGIS)

```python
from qgis.core import (
    QgsCoordinateReferenceSystem,
    QgsCoordinateTransform,
    QgsProject,
    QgsPointXY
)

# Define CRS objects
source_crs = QgsCoordinateReferenceSystem("EPSG:28992")
target_crs = QgsCoordinateReferenceSystem("EPSG:4326")

# Create transformation
xform = QgsCoordinateTransform(source_crs, target_crs, QgsProject.instance())

# Transform a point
result = xform.transform(QgsPointXY(155000, 463000))
# Returns approximately (5.387, 52.156)
```

---

## Datum Transformation Accuracy

| Method | Accuracy | Use When |
|--------|----------|----------|
| 7-parameter Helmert (TOWGS84) | 1–5 m | Quick approximation, no grid files available |
| Grid-based (NTv2 horizontal) | 1–10 cm | Production accuracy required |
| Grid-based (geoid vertical) | 1–5 cm | Vertical datum conversion needed |
| Same-datum reprojection | Float precision only | Converting between projections on same datum |
