# anti-patterns.md — Coordinate Handling Mistakes to Avoid

## Anti-Pattern 1: Assuming Georeferencing Exists

### Wrong

```python
# BROKEN: crashes if IfcMapConversion is absent
ifc = ifcopenshell.open("model.ifc")
mc = ifc.by_type("IfcMapConversion")[0]  # IndexError if no georeferencing
easting = mc.Eastings
```

### Correct

```python
ifc = ifcopenshell.open("model.ifc")
map_convs = ifc.by_type("IfcMapConversion")
if not map_convs:
    raise ValueError("IFC file has no georeferencing (IfcMapConversion missing)")
mc = map_convs[0]
easting = mc.Eastings
```

### Why
Many BIM authoring tools export IFC files without georeferencing by default. ALWAYS check for the presence of `IfcMapConversion` and `IfcProjectedCRS` before accessing their attributes.

---

## Anti-Pattern 2: Omitting always_xy=True in pyproj

### Wrong

```python
from pyproj import Transformer

t = Transformer.from_crs("EPSG:4326", "EPSG:28992")
# EPSG:4326 has native axis order (lat, lon) — NOT (lon, lat)
e, n = t.transform(4.9, 52.37)  # WRONG: 4.9 is treated as latitude
```

### Correct

```python
from pyproj import Transformer

t = Transformer.from_crs("EPSG:4326", "EPSG:28992", always_xy=True)
e, n = t.transform(4.9, 52.37)  # CORRECT: 4.9=longitude, 52.37=latitude
```

### Why
EPSG:4326 defines axis order as (latitude, longitude), but most developers expect (longitude, latitude). The `always_xy=True` flag forces consistent (x=lon, y=lat) ordering regardless of CRS definition. NEVER omit this parameter — silent coordinate swaps produce locations that are plausible but wrong.

---

## Anti-Pattern 3: Hardcoding Scale Factor

### Wrong

```python
# BROKEN: assumes model is in millimeters
map_conv.Scale = 0.001
```

### Correct

```python
# Read actual project units, then set scale
for ua in ifc.by_type("IfcUnitAssignment"):
    for unit in ua.Units:
        if hasattr(unit, "UnitType") and unit.UnitType == "LENGTHUNIT":
            if hasattr(unit, "Prefix") and unit.Prefix == "MILLI":
                scale = 0.001
            elif hasattr(unit, "Name") and unit.Name == "FOOT":
                scale = 0.3048
            else:
                scale = 1.0
```

### Why
Different BIM authoring tools use different internal units. Revit uses feet, FreeCAD uses millimeters, Blender uses meters. ALWAYS read `IfcUnitAssignment` to determine the actual project units before computing the scale factor.

---

## Anti-Pattern 4: Applying Y/Z Swap Universally

### Wrong

```javascript
// BROKEN: applies axis swap to ALL loaded models
loader.onLoad = (model) => {
    model.rotation.x = -Math.PI / 2;  // Always rotate
};
```

### Correct

```javascript
loader.onLoad = (model) => {
    // Check if the source data is Z-up (IFC, Blender) or Y-up (glTF)
    if (sourceFormat === "ifc" || sourceFormat === "obj") {
        model.rotation.x = -Math.PI / 2;  // Z-up → Y-up
    }
    // glTF is already Y-up — no rotation needed
};
```

### Why
Not all 3D formats use the same axis convention. glTF is Y-up by specification. IFC is Z-up. OBJ files vary. NEVER apply axis corrections without checking the source format. Applying a Y/Z swap to already-Y-up data rotates the model 90° in the wrong direction.

---

## Anti-Pattern 5: Ignoring IfcMapConversion Defaults

### Wrong

```python
mc = ifc.by_type("IfcMapConversion")[0]
xa = mc.XAxisAbscissa   # Could be None
xo = mc.XAxisOrdinate   # Could be None
angle = math.atan2(xo, xa)  # TypeError: can't use None in math
```

### Correct

```python
mc = ifc.by_type("IfcMapConversion")[0]
xa = mc.XAxisAbscissa if mc.XAxisAbscissa is not None else 1.0
xo = mc.XAxisOrdinate if mc.XAxisOrdinate is not None else 0.0
scale = mc.Scale if mc.Scale is not None else 1.0
angle = math.atan2(xo, xa)
```

### Why
IFC optional attributes return `None` when not set. The IFC specification defines default values (XAxisAbscissa=1.0, XAxisOrdinate=0.0, Scale=1.0), but IfcOpenShell does NOT apply these defaults automatically. ALWAYS provide fallback values when reading optional IfcMapConversion attributes.

---

## Anti-Pattern 6: Mixing Coordinate Systems in One Dataset

### Wrong

```python
# BROKEN: mixing RD New and UTM coordinates in one GeoDataFrame
buildings_rd = gpd.read_file("buildings_rd.shp")     # EPSG:28992
buildings_utm = gpd.read_file("buildings_utm.shp")   # EPSG:32631
all_buildings = pd.concat([buildings_rd, buildings_utm])  # Mixed CRS!
```

### Correct

```python
buildings_rd = gpd.read_file("buildings_rd.shp")     # EPSG:28992
buildings_utm = gpd.read_file("buildings_utm.shp")   # EPSG:32631

# Transform to common CRS BEFORE merging
buildings_utm_as_rd = buildings_utm.to_crs(epsg=28992)
all_buildings = pd.concat([buildings_rd, buildings_utm_as_rd])
all_buildings.crs = buildings_rd.crs  # Verify CRS is set
```

### Why
Concatenating geodata in different CRS produces geometrically invalid results. The numeric coordinates are incompatible — an easting of 155,000 means entirely different locations in RD New vs UTM 31N. ALWAYS transform all datasets to a single CRS before combining them.

---

## Anti-Pattern 7: Assuming Heights Are Ellipsoidal

### Wrong

```python
# BROKEN: passes NAP height directly to Cesium (expects ellipsoidal)
cesium_position = {
    "longitude": lon,
    "latitude": lat,
    "height": mc.OrthogonalHeight  # NAP height, NOT ellipsoidal
}
```

### Correct

```python
from pyproj import Transformer

# Transform NAP → ellipsoidal height
t = Transformer.from_crs("EPSG:7415", "EPSG:4979", always_xy=True)
lon, lat, h_ellipsoid = t.transform(easting, northing, mc.OrthogonalHeight)

cesium_position = {
    "longitude": lon,
    "latitude": lat,
    "height": h_ellipsoid  # Correct ellipsoidal height
}
```

### Why
BIM models store heights relative to a vertical datum (e.g., NAP in the Netherlands). 3D globe viewers (Cesium, Google Earth) use ellipsoidal heights. The difference is the geoid undulation — approximately 42–44 meters in the Netherlands. NEVER pass orthometric heights directly to a globe viewer.

---

## Anti-Pattern 8: Transforming Coordinates One at a Time

### Wrong

```python
from pyproj import Transformer

t = Transformer.from_crs("EPSG:28992", "EPSG:4326", always_xy=True)
results = []
for point in ten_million_points:
    lon, lat = t.transform(point.x, point.y)  # Slow: Python loop overhead
    results.append((lon, lat))
```

### Correct

```python
import numpy as np
from pyproj import Transformer

t = Transformer.from_crs("EPSG:28992", "EPSG:4326", always_xy=True)
xs = np.array([p.x for p in ten_million_points])
ys = np.array([p.y for p in ten_million_points])
lons, lats = t.transform(xs, ys)  # Vectorized: orders of magnitude faster
```

### Why
pyproj's `Transformer.transform()` accepts arrays and processes them in compiled C code. Calling it in a Python loop adds massive overhead for large datasets. ALWAYS pass arrays or lists to `transform()` instead of individual coordinate pairs.

---

## Anti-Pattern 9: Using Deprecated Proj4 Strings

### Wrong

```python
from pyproj import Proj

# DEPRECATED: Proj4 string, old API
p = Proj("+proj=sterea +lat_0=52.156 +lon_0=5.387 +k=0.9999079 ...")
x, y = p(lon, lat)
```

### Correct

```python
from pyproj import Transformer

# Use EPSG codes with the Transformer API
t = Transformer.from_crs("EPSG:4326", "EPSG:28992", always_xy=True)
x, y = t.transform(lon, lat)
```

### Why
The `Proj` class and raw Proj4 strings are deprecated in pyproj 2.x+. They lack proper datum transformation support and may silently produce less accurate results. ALWAYS use `Transformer.from_crs()` with EPSG codes — it automatically selects the best available transformation pipeline.

---

## Anti-Pattern 10: Fixing Errors in Wrong Order

### Wrong

```javascript
// BROKEN order: translate, then scale, then rotate
model.position.set(121687, 0, -487484);  // Translation first
model.scale.set(0.001, 0.001, 0.001);   // Then scale
model.rotation.x = -Math.PI / 2;        // Then rotate
// Result: translation is scaled down by 0.001, model at wrong position
```

### Correct

```javascript
// CORRECT order: scale, rotate, translate
model.scale.set(0.001, 0.001, 0.001);   // 1. Scale (mm → m)
model.rotation.x = -Math.PI / 2;        // 2. Axis swap (Z-up → Y-up)
model.position.set(121687, 0, -487484); // 3. Position (in target units)
model.updateMatrixWorld(true);
```

### Why
Three.js applies transformations in the order: scale → rotation → translation (SRT). Setting properties in a different order still applies them as SRT. However, if you compute translation values assuming pre-scaled coordinates, you must set the translation AFTER determining the post-scale coordinate space. ALWAYS apply corrections in order: unit conversion (scale) first, axis swap (rotation) second, georeferencing offset (translation) last.
