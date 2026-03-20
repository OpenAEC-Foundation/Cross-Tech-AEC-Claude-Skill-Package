# anti-patterns.md — Coordinate System Mistakes

## Anti-Pattern 1: Missing `always_xy=True` in pyproj

**Symptom**: Coordinates are silently swapped — easting and northing exchange values, placing the model in the wrong location.

**Why it happens**: EPSG:4326 (WGS84) officially defines axis order as (latitude, longitude), not (longitude, latitude). Without `always_xy=True`, pyproj follows the official order.

**Wrong**:
```python
transformer = Transformer.from_crs("EPSG:4326", "EPSG:28992")
easting, northing = transformer.transform(4.9003, 52.3792)
# WRONG — pyproj interprets 4.9003 as latitude, 52.3792 as longitude
```

**Correct**:
```python
transformer = Transformer.from_crs("EPSG:4326", "EPSG:28992", always_xy=True)
easting, northing = transformer.transform(4.9003, 52.3792)
# CORRECT — 4.9003 is longitude (x), 52.3792 is latitude (y)
```

**Rule**: ALWAYS pass `always_xy=True` to `Transformer.from_crs()`.

---

## Anti-Pattern 2: Importing BIM into GIS Without Georeferencing Check

**Symptom**: Model appears at (0°N, 0°E) — in the Gulf of Guinea, off the coast of West Africa.

**Why it happens**: When IfcMapConversion and IfcProjectedCRS are absent, local coordinates (0,0,0) are interpreted as geographic (0,0) — the intersection of the Equator and Prime Meridian.

**Wrong**:
```python
# Directly use local coordinates as map coordinates
layer = QgsVectorLayer(f"Point?crs=EPSG:28992", "BIM", "memory")
feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(local_x, local_y)))
# Model appears near (0, 0) — WRONG
```

**Correct**:
```python
mc = ifc.by_type("IfcMapConversion")
if not mc:
    raise ValueError("No georeferencing — cannot place on map")

e, n, h = geo.auto_xyz2enh(ifc, local_x, local_y, local_z)
feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(e, n)))
```

**Rule**: ALWAYS check for IfcMapConversion before converting BIM coordinates to GIS.

---

## Anti-Pattern 3: Wrong EPSG Code Assignment

**Symptom**: Model appears hundreds of kilometers away from its actual location.

**Why it happens**: Using EPSG:32631 (UTM zone 31N) when coordinates are actually in EPSG:28992 (RD New), or vice versa. Both use meters, but their origins and projections differ.

**Detection**:
```python
CRS_RANGES = {
    "EPSG:28992": {"e_min": -7000, "e_max": 300000, "n_min": 289000, "n_max": 629000},
    "EPSG:32631": {"e_min": 166000, "e_max": 834000, "n_min": 0, "n_max": 9400000},
}

# If easting=155000, northing=463000 → matches RD New, NOT UTM 31N
```

**Rule**: ALWAYS validate that coordinate values fall within the valid range for the assigned CRS.

---

## Anti-Pattern 4: Y-Up / Z-Up Axis Mismatch

**Symptom**: Model appears rotated 90° — lying on its side in the web viewer.

**Why it happens**: BIM tools (IFC, Blender, QGIS, Revit, FreeCAD) use Z-up. Three.js and Babylon.js use Y-up. Failing to swap axes produces a sideways model.

**Wrong**:
```javascript
// Directly using BIM coordinates in Three.js
mesh.position.set(bim_x, bim_y, bim_z);  // Model lies on its side
```

**Correct**:
```javascript
mesh.position.set(bim_x, bim_z, -bim_y);  // Proper Y/Z swap with negation
```

**Rule**: ALWAYS apply the swap `(x, z, -y)` when moving from Z-up to Y-up systems. The negation preserves right-handedness.

---

## Anti-Pattern 5: Unit Mismatch Between IFC and CRS

**Symptom**: Building appears 1000x too large or ~0.3x too small on the map.

**Why it happens**: IFC model uses millimeters but CRS expects meters (or Revit feet to metric CRS), and `IfcMapConversion.Scale` is missing or set to 1.0.

| Scenario | Visual Effect | Required Scale |
|----------|--------------|----------------|
| IFC in mm, CRS in m, Scale=1.0 | 1000x too large | Scale should be 0.001 |
| Revit feet, CRS in m, Scale=1.0 | ~0.3x too small | Scale should be 0.3048 |
| IFC in m, CRS in m, Scale=1.0 | Correct | No change needed |

**Wrong**:
```python
# Ignoring model units
e, n, h = geo.xyz2enh(x_mm, y_mm, z_mm,
    eastings=155000, northings=463000, orthogonal_height=0,
    scale=1.0)  # BUG — model is in mm, CRS in meters
```

**Correct**:
```python
# Check project units first
units = ifc.by_type("IfcUnitAssignment")[0]
# Determine if model is in mm, m, or feet
# Set scale accordingly
e, n, h = geo.xyz2enh(x_mm, y_mm, z_mm,
    eastings=155000, northings=463000, orthogonal_height=0,
    scale=0.001)  # mm → m conversion
```

**Rule**: ALWAYS verify project units from IfcUnitAssignment and ensure Scale converts correctly to CRS map units.

---

## Anti-Pattern 6: Mixing UTM Zones

**Symptom**: Points from two datasets that should be adjacent appear hundreds of kilometers apart.

**Why it happens**: UTM zone boundaries create a discontinuity. Zone 31N and Zone 32N have overlapping valid coordinate ranges but with completely different eastings for the same physical location.

**Wrong**:
```python
# Dataset A in UTM 31N, Dataset B in UTM 32N
# Treating both as same CRS
combined_eastings = np.concatenate([eastings_31n, eastings_32n])  # WRONG
```

**Correct**:
```python
# Reproject everything to a single CRS first
t = Transformer.from_crs("EPSG:32632", "EPSG:32631", always_xy=True)
eastings_31n_from_32n, northings_31n_from_32n = t.transform(
    eastings_32n, northings_32n
)
```

**Rule**: NEVER combine coordinates from different UTM zones. Reproject to a single CRS first.

---

## Anti-Pattern 7: Confusing NAP Height with Ellipsoidal Height

**Symptom**: Buildings float ~42 meters above terrain in 3D viewers (Netherlands).

**Why it happens**: NAP (Normaal Amsterdams Peil) is a physical datum tied to mean sea level. WGS84 ellipsoidal height measures from the mathematical ellipsoid. In the Netherlands, the geoid undulation (difference) is approximately 42–44 meters.

**Wrong**:
```python
# Using NAP height directly as WGS84 ellipsoidal height
position_wgs84 = (lon, lat, nap_height)  # Building floats 42m above terrain
```

**Correct**:
```python
# Use compound CRS for proper vertical conversion
transformer = Transformer.from_crs(
    "EPSG:7415",   # RD New + NAP
    "EPSG:4979",   # WGS84 3D (with ellipsoidal height)
    always_xy=True
)
lon, lat, ellipsoidal_h = transformer.transform(easting, northing, nap_height)
```

**Rule**: ALWAYS use a compound CRS (e.g., EPSG:7415) and proper vertical datum transformation when converting between NAP and ellipsoidal heights.

---

## Anti-Pattern 8: True North Rotation Error

**Symptom**: Building appears rotated on the map — walls do not align with roads/parcels.

**Why it happens**: BIM models often use Project North (aligned with building grid) rather than True North. The rotation must be stored in XAxisAbscissa/XAxisOrdinate. Wrong values cause the building to be rotated incorrectly.

**Wrong**:
```python
# Setting rotation in degrees instead of cos/sin components
coordinate_operation={
    "XAxisAbscissa": 30.0,  # WRONG — this is NOT degrees
    "XAxisOrdinate": 0.0,
}
```

**Correct**:
```python
import math
angle_rad = math.radians(30.0)
coordinate_operation={
    "XAxisAbscissa": math.cos(angle_rad),  # 0.866025
    "XAxisOrdinate": math.sin(angle_rad),   # 0.5
}
```

**Rule**: ALWAYS encode true north rotation as `cos(θ)` and `sin(θ)` in XAxisAbscissa and XAxisOrdinate respectively. The vector MUST have unit length.

---

## Anti-Pattern 9: Hardcoding TOWGS84 Parameters

**Symptom**: Transformation accuracy degrades, or transformation fails in edge cases.

**Why it happens**: Manually specifying datum transformation parameters bypasses PROJ's ability to select the best available transformation (including grid-based high-accuracy methods) for the specific geographic area.

**Wrong**:
```python
# Hardcoded 7-parameter Helmert — 1-5m accuracy only
crs_rd = CRS.from_proj4(
    "+proj=sterea +lat_0=52.156160556 +lon_0=5.387638889 "
    "+k=0.9999079 +x_0=155000 +y_0=463000 +ellps=bessel "
    "+towgs84=565.4171,50.3319,465.5524,1.9342,-1.6677,9.1019,4.0725"
)
```

**Correct**:
```python
# Let PROJ select the best transformation automatically
transformer = Transformer.from_crs("EPSG:28992", "EPSG:4326", always_xy=True)
# PROJ will use grid-based RDNAPTRANS if available (~1cm accuracy)
```

**Rule**: NEVER hardcode TOWGS84 parameters. Use EPSG codes and let PROJ select the optimal transformation path.

---

## Anti-Pattern 10: Using EPSG:3857 for Engineering Work

**Symptom**: Distances and areas are distorted, especially at higher latitudes.

**Why it happens**: EPSG:3857 (Web Mercator) is designed for web map display, not measurement. It distorts both area and distance, increasingly at higher latitudes. At 52°N (Netherlands), the scale distortion is approximately 1.63x.

**Wrong**:
```python
# Measuring building dimensions in Web Mercator
transformer = Transformer.from_crs("EPSG:4326", "EPSG:3857", always_xy=True)
x1, y1 = transformer.transform(lon1, lat1)
x2, y2 = transformer.transform(lon2, lat2)
distance = math.sqrt((x2-x1)**2 + (y2-y1)**2)  # WRONG — distorted
```

**Correct**:
```python
# Use a local projected CRS for measurements
transformer = Transformer.from_crs("EPSG:4326", "EPSG:28992", always_xy=True)
x1, y1 = transformer.transform(lon1, lat1)
x2, y2 = transformer.transform(lon2, lat2)
distance = math.sqrt((x2-x1)**2 + (y2-y1)**2)  # Correct in meters
```

**Rule**: NEVER use EPSG:3857 for distance or area calculations. Use a local projected CRS (e.g., EPSG:28992 for NL) for engineering measurements.
