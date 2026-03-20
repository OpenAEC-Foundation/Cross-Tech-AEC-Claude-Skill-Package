# examples.md — Real Coordinate Error Scenarios with Fixes

## Example 1: IFC Model in Gulf of Guinea (Missing Georeferencing)

### Scenario
A structural engineer exports a Revit model to IFC and sends it to a GIS team. When loaded in QGIS, the building appears at coordinates (0°N, 0°E) — in the Gulf of Guinea, off the coast of West Africa.

### Root Cause
The Revit export did not include georeferencing. The IFC file has no `IfcMapConversion` or `IfcProjectedCRS`. QGIS interprets the local coordinates as map coordinates in the project CRS, placing the model at the origin.

### Diagnosis

```python
import ifcopenshell

ifc = ifcopenshell.open("structure.ifc")

# Step 1: Check for georeferencing entities
print(f"IfcMapConversion: {len(ifc.by_type('IfcMapConversion'))}")  # 0
print(f"IfcProjectedCRS: {len(ifc.by_type('IfcProjectedCRS'))}")    # 0

# Step 2: Check coordinate range of geometry
products = ifc.by_type("IfcProduct")
for p in products[:5]:
    placement = p.ObjectPlacement
    if placement and hasattr(placement, "RelativePlacement"):
        loc = placement.RelativePlacement.Location
        if loc:
            print(f"{p.Name}: ({loc[0]}, {loc[1]}, {loc[2]})")
# Output: local coordinates like (5.0, 12.0, 0.0) — clearly not map coordinates
```

### Fix

```python
import ifcopenshell
import ifcopenshell.api

ifc = ifcopenshell.open("structure.ifc")

# Add georeferencing for a Dutch project in RD New
ifcopenshell.api.run("georeference.add_georeferencing", ifc)
ifcopenshell.api.run("georeference.edit_georeferencing", ifc,
    projected_crs={"Name": "EPSG:28992"},
    coordinate_operation={
        "Eastings": 121687.0,     # From site survey
        "Northings": 487484.0,    # From site survey
        "OrthogonalHeight": -1.5, # NAP height of project origin
    })

ifc.write("structure_georef.ifc")
```

### Verification
After adding georeferencing, reload in QGIS. The building should appear in Amsterdam (approximately).

---

## Example 2: Building 1000x Too Large on Map (mm vs m)

### Scenario
An architect exports an IFC model from ArchiCAD (project units: millimeters). When loaded in QGIS with CRS set to EPSG:28992, the building footprint spans several kilometers.

### Root Cause
The IFC file has `IfcMapConversion` with `Scale = None` (defaults to 1.0). Since the project uses millimeters but the CRS uses meters, all coordinates are 1000x too large.

### Diagnosis

```python
import ifcopenshell

ifc = ifcopenshell.open("archicad_model.ifc")

# Check project units
for ua in ifc.by_type("IfcUnitAssignment"):
    for unit in ua.Units:
        if hasattr(unit, "UnitType") and unit.UnitType == "LENGTHUNIT":
            prefix = getattr(unit, "Prefix", None)
            print(f"Length unit: {unit.Name}, prefix: {prefix}")
            # Output: Length unit: METRE, prefix: MILLI

# Check scale
mc = ifc.by_type("IfcMapConversion")[0]
print(f"Scale: {mc.Scale}")  # Output: Scale: None (defaults to 1.0)
# BUG: should be 0.001 to convert mm → m
```

### Fix

```python
import ifcopenshell
import ifcopenshell.api

ifc = ifcopenshell.open("archicad_model.ifc")

mc = ifc.by_type("IfcMapConversion")[0]
ifcopenshell.api.run("georeference.edit_georeferencing", ifc,
    coordinate_operation={
        "Eastings": mc.Eastings,
        "Northings": mc.Northings,
        "OrthogonalHeight": mc.OrthogonalHeight,
        "Scale": 0.001,  # mm → m conversion
    })

ifc.write("archicad_model_fixed.ifc")
```

---

## Example 3: Model Lying on Its Side in Three.js (Y/Z Swap)

### Scenario
A developer loads an IFC file using web-ifc directly (not through ThatOpen Components). The building appears rotated 90° — floors are vertical, walls are horizontal.

### Root Cause
IFC uses Z-up convention. Three.js uses Y-up. The raw IFC coordinates were used without axis transformation.

### Diagnosis

```javascript
import * as THREE from 'three';

// After loading IFC geometry into Three.js
const box = new THREE.Box3().setFromObject(ifcModel);
const size = box.getSize(new THREE.Vector3());

console.log(`Size: X=${size.x.toFixed(1)}, Y=${size.y.toFixed(1)}, Z=${size.z.toFixed(1)}`);
// Output: Size: X=30.0, Y=0.2, Z=12.5
// Y (Three.js up) is only 0.2m — this is the slab thickness
// Z is 12.5m — this is the building height, wrongly in the depth axis
```

### Fix

```javascript
// Option A: Rotate the root object
ifcModel.rotation.x = -Math.PI / 2;

// Option B: Transform individual vertices (more control)
const positions = geometry.attributes.position;
for (let i = 0; i < positions.count; i++) {
    const x = positions.getX(i);
    const y = positions.getY(i);
    const z = positions.getZ(i);
    positions.setXYZ(i, x, z, -y);  // BIM Z→Three.js Y, BIM -Y→Three.js Z
}
positions.needsUpdate = true;
```

---

## Example 4: Building Rotated 30° on Map (True North Error)

### Scenario
A BIM model placed in QGIS appears at the correct location, but the building is rotated approximately 30°. The building grid does not align with the street grid visible on the base map.

### Root Cause
In Revit, True North was set 30° from Project North. The IFC export captured this as `XAxisAbscissa=0.866` and `XAxisOrdinate=0.5`. However, the GIS import tool ignored these rotation parameters.

### Diagnosis

```python
import math
import ifcopenshell

ifc = ifcopenshell.open("rotated_model.ifc")
mc = ifc.by_type("IfcMapConversion")[0]

xa = mc.XAxisAbscissa or 1.0
xo = mc.XAxisOrdinate or 0.0
angle = math.degrees(math.atan2(xo, xa))
print(f"True North rotation: {angle:.1f}°")  # Output: 30.0°
```

### Fix
If the GIS tool does not apply the rotation, pre-rotate coordinates:

```python
import math
import ifcopenshell
import ifcopenshell.util.geolocation as geo

ifc = ifcopenshell.open("rotated_model.ifc")

# Convert all product placements to map coordinates (includes rotation)
for product in ifc.by_type("IfcProduct"):
    if product.ObjectPlacement:
        matrix = ifcopenshell.util.placement.get_local_placement(
            product.ObjectPlacement)
        global_matrix = geo.auto_local2global(ifc, matrix)
        # Use global_matrix for GIS placement — rotation is applied
```

---

## Example 5: Heights Off by 43m (NAP vs Ellipsoid)

### Scenario
A BIM model of a building at ground level in Amsterdam appears floating 43 meters above the terrain in a Cesium 3D Tiles viewer. Horizontal position is correct.

### Root Cause
The IFC model uses NAP heights (orthometric, Dutch vertical datum). The 3D Tiles viewer expects WGS84 ellipsoidal heights. In Amsterdam, the geoid undulation (ellipsoid minus geoid) is approximately +43m.

### Diagnosis

```python
import ifcopenshell

ifc = ifcopenshell.open("amsterdam_building.ifc")
crs = ifc.by_type("IfcProjectedCRS")[0]
mc = ifc.by_type("IfcMapConversion")[0]

print(f"Vertical datum: {crs.VerticalDatum}")      # Output: "NAP" or None
print(f"Orthogonal height: {mc.OrthogonalHeight}")  # Output: -0.5 (NAP)
```

### Fix

```python
from pyproj import Transformer

# Transform from RD New + NAP to WGS84 3D (ellipsoidal heights)
t = Transformer.from_crs("EPSG:7415", "EPSG:4979", always_xy=True)

# Building corner at NAP -0.5m
lon, lat, h_ellipsoid = t.transform(121687.0, 487484.0, -0.5)
print(f"Ellipsoidal height: {h_ellipsoid:.1f}m")  # ~42.5m
# The "correct" height for the Cesium viewer is ~42.5m above the ellipsoid
```

---

## Example 6: Wrong CRS — RD New Coordinates Treated as UTM

### Scenario
An IFC file has `IfcProjectedCRS.Name = "EPSG:32631"` (UTM zone 31N), but the actual coordinates are in EPSG:28992 (RD New). The model appears ~200km north of the expected location.

### Diagnosis

```python
import ifcopenshell

ifc = ifcopenshell.open("wrong_crs.ifc")
crs = ifc.by_type("IfcProjectedCRS")[0]
mc = ifc.by_type("IfcMapConversion")[0]

print(f"CRS: {crs.Name}")          # EPSG:32631
print(f"E: {mc.Eastings}")          # 155000.0
print(f"N: {mc.Northings}")         # 463000.0

# Validate: RD New range check
e, n = mc.Eastings, mc.Northings
if -7000 <= e <= 300000 and 289000 <= n <= 629000:
    print("Coordinates match EPSG:28992 (RD New) range, NOT UTM 31N")
# UTM 31N eastings typically range 166,000-834,000
# RD New eastings range -7,000-300,000
# Value 155,000 fits RD New but is borderline for UTM 31N
```

### Fix

```python
import ifcopenshell
import ifcopenshell.api

ifc = ifcopenshell.open("wrong_crs.ifc")

ifcopenshell.api.run("georeference.edit_georeferencing", ifc,
    projected_crs={"Name": "EPSG:28992"})  # Correct CRS

ifc.write("correct_crs.ifc")
```

---

## Example 7: Compound Error — Scale + Missing Georef + Axis Swap

### Scenario
A developer loads a FreeCAD IFC export (mm units, no georeferencing) into a Three.js web viewer. The model is invisible — no geometry appears in the viewport.

### Root Cause
Three compound errors:
1. No georeferencing → coordinates near (0, 0, 0)
2. FreeCAD exports in mm → a 10m wall has coordinate value 10000
3. No Y/Z axis swap → model lies flat in Three.js

The model exists but is 1000x too large and lying on its side, extending far beyond the default camera frustum.

### Diagnosis

```javascript
const box = new THREE.Box3().setFromObject(scene);
console.log("Min:", box.min.toArray());  // [0, 0, -50000]
console.log("Max:", box.max.toArray());  // [30000, 200, 0]
// X span: 30,000 (should be ~30 for a 30m building in meters)
// Y span: 200 (should be building height ~12m, but this is slab thickness in mm)
// Z span: 50,000 (building depth in mm, negative = no axis swap)
```

### Fix (apply in order)

```javascript
// Step 1: Scale mm → m
model.scale.set(0.001, 0.001, 0.001);

// Step 2: Axis swap Z-up → Y-up
model.rotation.x = -Math.PI / 2;

// Step 3: Position at world origin (or apply georeferencing offset)
model.position.set(0, 0, 0);

// Step 4: Update and fit camera
model.updateMatrixWorld(true);
const box = new THREE.Box3().setFromObject(model);
const center = box.getCenter(new THREE.Vector3());
camera.position.copy(center).add(new THREE.Vector3(50, 50, 50));
camera.lookAt(center);
```

**ALWAYS** fix errors in this order: scale first, then axis swap, then translation. Reversing the order produces incorrect results.
