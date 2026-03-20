---
name: crosstech-errors-coordinate-mismatch
description: >
  Use when BIM models appear at the wrong location, wrong scale, or wrong orientation
  in GIS or web viewers. Provides a diagnostic decision tree for coordinate-related
  errors: Y-up vs Z-up axis swap, unit mismatches (mm/m/ft), missing georeferencing,
  CRS mismatches, and datum transformation errors.
  Covers coordinate debugging across Blender, Three.js, QGIS, IFC, Revit, and FreeCAD.
  Keywords: wrong location, coordinate mismatch, Y-up Z-up, scale error, EPSG,
  georeferencing missing, axis swap, unit mismatch, CRS error, datum shift.
license: MIT
compatibility: "Designed for Claude Code. Covers all AEC tools with coordinate systems."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-errors-coordinate-mismatch

## Quick Reference

| Symptom | Likely Cause | Jump To |
|---------|-------------|---------|
| Model rotated 90° / lying on side | Y-up vs Z-up axis swap | [Failure Mode 1](#failure-mode-1-y-up-vs-z-up-axis-swap) |
| Model 1000x too large or too small | Unit mismatch (mm/m/ft) | [Failure Mode 2](#failure-mode-2-unit-mismatch-mmmft) |
| Model at (0°N, 0°E) in Gulf of Guinea | Missing georeferencing | [Failure Mode 3](#failure-mode-3-missing-georeferencing) |
| Model offset by hundreds of km | Wrong CRS / EPSG code | [Failure Mode 4](#failure-mode-4-wrong-crs--epsg-code) |
| Building rotated on map, footprint correct | True North rotation error | [Failure Mode 5](#failure-mode-5-true-north-rotation-error) |
| Coordinates jump at project boundary | UTM zone boundary | [Failure Mode 6](#failure-mode-6-utm-zone-boundary) |
| Building floating ~43m above terrain | Vertical datum confusion | [Failure Mode 7](#failure-mode-7-vertical-datum-confusion) |

---

## Axis Convention Table

| Tool | Up Axis | Handedness | Default Units | Coordinate Order |
|------|---------|-----------|---------------|-----------------|
| **Blender** | Z-up | Right-handed | Meters | (X, Y, Z) |
| **Three.js** | Y-up | Right-handed | Unitless | (X, Y, Z) |
| **QGIS** | Z-up (2.5D) | Right-handed | Map units (m/deg) | (X=East, Y=North) |
| **IFC** | Z-up | Right-handed | Project units (mm/m) | (X, Y, Z) |
| **Revit** | Z-up | Right-handed | Internal: feet | (X, Y, Z) |
| **FreeCAD** | Z-up | Right-handed | Internal: mm | (X, Y, Z) |
| **Speckle** | Z-up | Right-handed | Meters | (X, Y, Z) |
| **web-ifc** | Z-up | Right-handed | IFC project units | (X, Y, Z) |

**Key boundary**: ALL BIM/GIS tools use Z-up. ONLY web 3D renderers (Three.js, Babylon.js) use Y-up.

---

## Diagnostic Decision Tree

```
START: Model appears wrong in target application
│
├─ Is the model rotated 90°?
│  YES → Failure Mode 1 (Y/Z axis swap)
│
├─ Is the model at the correct location but wrong size?
│  ├─ ~1000x too large → IFC in mm, target expects m (Scale missing)
│  ├─ ~3.28x too large → IFC in feet, target expects m
│  ├─ ~0.3x too small  → IFC in m, target expects feet
│  └─ YES → Failure Mode 2 (unit mismatch)
│
├─ Is the model near (0, 0) / Gulf of Guinea?
│  YES → Failure Mode 3 (missing georeferencing)
│
├─ Is the model offset by >1 km from expected position?
│  ├─ Offset ~100–500 km → Failure Mode 4 (wrong CRS)
│  ├─ Offset ~500,000 m in easting → Failure Mode 6 (UTM zone)
│  └─ Offset < 5 m → Check datum transformation accuracy
│
├─ Is the model at the right position but rotated?
│  YES → Failure Mode 5 (True North rotation)
│
├─ Is the model at correct X/Y but wrong height (~40-44m off)?
│  YES → Failure Mode 7 (vertical datum)
│
└─ None of the above → Check for compound errors (multiple failures)
```

---

## Failure Mode 1: Y-Up vs Z-Up Axis Swap

### Symptom
Model appears rotated 90° — buildings lie on their side. Walls are horizontal, floors are vertical.

### Cause
BIM tools (IFC, Blender, Revit, QGIS) use Z-up. Web 3D renderers (Three.js, Babylon.js) use Y-up. Importing BIM data into a web viewer without axis transformation produces a 90° rotation.

### Detection

```javascript
// Three.js: check if bounding box height is near zero
const box = new THREE.Box3().setFromObject(model);
const size = box.getSize(new THREE.Vector3());
if (size.y < 0.1 && size.z > 1.0) {
    console.error("Y/Z axis swap detected: model is lying flat");
}
```

### Fix

```javascript
// JavaScript: BIM Z-up → Three.js Y-up
function bimToThreeJs(x, y, z) {
    return { x: x, y: z, z: -y };
}
```

The negation of `y` → `-z` preserves right-handedness. Without negation, the model appears mirrored.

**ALWAYS** apply this transform when loading IFC/BIM data into Three.js or Babylon.js.

**NEVER** apply this transform when loading into Blender, QGIS, or other Z-up tools.

### Tool-Specific Notes

- **web-ifc / ThatOpen Components**: Handles Y/Z swap automatically when using `FragmentsManager`. Manual loading via `IfcLoader` requires explicit swap.
- **Three.js GLTFLoader**: glTF is Y-up by spec — no swap needed for glTF files. Only swap for raw IFC coordinate data.
- **Speckle**: Converts axes automatically per target application on receive.

---

## Failure Mode 2: Unit Mismatch (mm/m/ft)

### Symptom
Model appears at the correct location but is absurdly large or small. A 10m building spans 10km on the map (mm→m) or appears as a 3m speck (feet→m without conversion).

### Detection

```python
import ifcopenshell

ifc = ifcopenshell.open("model.ifc")

# Step 1: Read project length units
units = ifc.by_type("IfcUnitAssignment")[0]
for unit in units.Units:
    if hasattr(unit, "UnitType") and unit.UnitType == "LENGTHUNIT":
        if hasattr(unit, "Prefix") and unit.Prefix == "MILLI":
            project_unit = "mm"
            expected_scale = 0.001
        elif hasattr(unit, "Name") and unit.Name == "FOOT":
            project_unit = "ft"
            expected_scale = 0.3048
        else:
            project_unit = "m"
            expected_scale = 1.0

# Step 2: Read IfcMapConversion Scale
map_conv = ifc.by_type("IfcMapConversion")
if map_conv:
    actual_scale = map_conv[0].Scale or 1.0  # None defaults to 1.0
    if abs(actual_scale - expected_scale) > 0.0001:
        print(f"SCALE MISMATCH: project={project_unit}, "
              f"MapConversion.Scale={actual_scale}, expected={expected_scale}")
```

### Common Scale Factors

| From | To | Scale Factor |
|------|----|-------------|
| mm | m | 0.001 |
| m | m | 1.0 |
| ft | m | 0.3048 |
| in | m | 0.0254 |

### Fix
Set `IfcMapConversion.Scale` to the correct conversion factor from project units to CRS map units.

**ALWAYS** check `IfcUnitAssignment` before interpreting `IfcMapConversion.Scale`.

**NEVER** assume project units are meters — Revit uses feet internally, FreeCAD uses mm.

---

## Failure Mode 3: Missing Georeferencing

### Symptom
Model appears at latitude 0°, longitude 0° (Gulf of Guinea, West Africa) or at the CRS origin point. This is the single most common BIM↔GIS integration failure.

### Detection

```python
import ifcopenshell

ifc = ifcopenshell.open("model.ifc")
has_map_conv = len(ifc.by_type("IfcMapConversion")) > 0
has_crs = len(ifc.by_type("IfcProjectedCRS")) > 0

if not has_map_conv:
    print("MISSING: IfcMapConversion — no coordinate transformation defined")
if not has_crs:
    print("MISSING: IfcProjectedCRS — no target CRS defined")
```

### Fix
Add georeferencing using IfcOpenShell:

```python
import ifcopenshell
import ifcopenshell.api

ifc = ifcopenshell.open("model.ifc")

# Add georeferencing entities
ifcopenshell.api.run("georeference.add_georeferencing", ifc)

# Set CRS and map conversion parameters
ifcopenshell.api.run("georeference.edit_georeferencing", ifc,
    projected_crs={"Name": "EPSG:28992"},
    coordinate_operation={
        "Eastings": 155000.0,
        "Northings": 463000.0,
        "OrthogonalHeight": 0.0,
    })

ifc.write("model_georef.ifc")
```

**ALWAYS** verify Eastings/Northings fall within the valid range of the target CRS.

**NEVER** leave IfcProjectedCRS.Name empty — downstream tools rely on this field to resolve the CRS.

---

## Failure Mode 4: Wrong CRS / EPSG Code

### Symptom
Model appears offset by tens to hundreds of kilometers from the expected position.

### Detection
Validate that coordinate values are plausible for the assigned CRS:

| CRS | Valid Easting Range | Valid Northing Range |
|-----|-------------------|---------------------|
| EPSG:28992 (RD New) | -7,000 to 300,000 | 289,000 to 629,000 |
| EPSG:32631 (UTM 31N) | 166,000 to 834,000 | 0 to 9,400,000 |
| EPSG:32632 (UTM 32N) | 166,000 to 834,000 | 0 to 9,400,000 |

```python
import ifcopenshell

ifc = ifcopenshell.open("model.ifc")
map_conv = ifc.by_type("IfcMapConversion")
crs = ifc.by_type("IfcProjectedCRS")

if map_conv and crs:
    e = map_conv[0].Eastings
    n = map_conv[0].Northings
    crs_name = crs[0].Name

    # Example: validate RD New range
    if "28992" in (crs_name or ""):
        if not (-7000 <= e <= 300000 and 289000 <= n <= 629000):
            print(f"COORDINATES OUT OF RANGE for {crs_name}: "
                  f"E={e}, N={n}")
```

### Fix
1. Determine the correct CRS from project documentation or site survey data
2. Verify with a known reference point (e.g., a building corner with known GPS coordinates)
3. Use pyproj to cross-check:

```python
from pyproj import Transformer

# Transform a known GPS point to the claimed CRS
t = Transformer.from_crs("EPSG:4326", "EPSG:28992", always_xy=True)
expected_e, expected_n = t.transform(lon, lat)  # always_xy: (lon, lat) order

print(f"Expected: E={expected_e:.1f}, N={expected_n:.1f}")
print(f"Actual:   E={map_conv[0].Eastings}, N={map_conv[0].Northings}")
```

**ALWAYS** use `always_xy=True` with pyproj — without it, axis order depends on CRS definition and causes silent coordinate swaps.

---

## Failure Mode 5: True North Rotation Error

### Symptom
Building footprint is at the correct map location but the building is rotated. Walls that should be parallel to streets appear at an angle.

### Detection

```python
import math
import ifcopenshell

ifc = ifcopenshell.open("model.ifc")
map_conv = ifc.by_type("IfcMapConversion")

if map_conv:
    xa = map_conv[0].XAxisAbscissa or 1.0
    xo = map_conv[0].XAxisOrdinate or 0.0
    angle_rad = math.atan2(xo, xa)
    angle_deg = math.degrees(angle_rad)
    vec_length = math.sqrt(xa**2 + xo**2)

    print(f"True North rotation: {angle_deg:.2f}°")
    if abs(vec_length - 1.0) > 0.001:
        print(f"WARNING: direction vector not normalized (length={vec_length:.4f})")
```

### Fix
Set `XAxisAbscissa = cos(angle)` and `XAxisOrdinate = sin(angle)` where `angle` is the anti-clockwise rotation from grid north (CRS Y-axis) to project north.

**ALWAYS** verify the direction vector is normalized (length = 1.0).

**NEVER** confuse the sign convention — positive angle = anti-clockwise rotation in IFC.

---

## Failure Mode 6: UTM Zone Boundary

### Symptom
Project straddles a UTM zone boundary. Coordinates jump by ~500,000m in easting, or measurements near the boundary show 0.04% distortion.

### Detection

```python
from pyproj import CRS

def utm_zone_from_longitude(lon):
    """Return UTM zone number for a given longitude."""
    return int((lon + 180) / 6) + 1

# Example: Netherlands straddles UTM zones 31 and 32
# Zone 31N: 0°E to 6°E (central meridian 3°E)
# Zone 32N: 6°E to 12°E (central meridian 9°E)
western_nl_zone = utm_zone_from_longitude(4.5)   # Zone 31
eastern_nl_zone = utm_zone_from_longitude(6.5)   # Zone 32
```

### Fix
**NEVER** mix coordinates from different UTM zones in one project.

Use a single CRS that covers the entire project area:
- Netherlands: EPSG:28992 (RD New) covers the entire country
- Europe-wide: EPSG:3035 (ETRS89 / LAEA) for pan-European projects
- If UTM is required, pick the zone containing the project centroid

---

## Failure Mode 7: Vertical Datum Confusion

### Symptom
Model floats ~42–44 meters above the terrain (Netherlands) or is buried below ground. Horizontal position is correct.

### Cause
Mixing NAP (Normaal Amsterdams Peil) orthometric heights with WGS84 ellipsoidal heights. In the Netherlands, the geoid undulation (difference between ellipsoid and geoid) is approximately 42–44 meters.

### Detection

```python
import ifcopenshell

ifc = ifcopenshell.open("model.ifc")
crs = ifc.by_type("IfcProjectedCRS")
map_conv = ifc.by_type("IfcMapConversion")

if crs:
    vert_datum = crs[0].VerticalDatum
    if vert_datum is None:
        print("WARNING: No vertical datum specified — height interpretation ambiguous")

if map_conv:
    h = map_conv[0].OrthogonalHeight
    if abs(h) > 40 and abs(h) < 50:
        print(f"SUSPICIOUS: OrthogonalHeight={h}m — possible NAP↔ellipsoid confusion")
```

### Fix

```python
from pyproj import Transformer

# Transform NAP height to WGS84 ellipsoidal height (or vice versa)
# Requires PROJ grid files (nl_nsgi_nlgeo2018.tif)
t = Transformer.from_crs(
    "EPSG:7415",   # RD New + NAP (compound)
    "EPSG:4979",   # WGS84 3D (geographic + ellipsoidal height)
    always_xy=True
)
lon, lat, h_ellipsoid = t.transform(easting, northing, h_nap)
geoid_undulation = h_ellipsoid - h_nap  # ~42-44m in NL
```

**ALWAYS** specify `IfcProjectedCRS.VerticalDatum` (e.g., "NAP") to make height interpretation unambiguous.

**NEVER** assume heights are ellipsoidal — most BIM models use orthometric (above sea level) heights.

---

## Critical Rules

1. **ALWAYS** check axis conventions before crossing a tool boundary — Z-up to Y-up swaps are the most common error.
2. **ALWAYS** verify `IfcMapConversion.Scale` matches the ratio of project units to CRS units.
3. **ALWAYS** use `always_xy=True` when creating pyproj Transformers.
4. **ALWAYS** validate coordinate ranges against the assigned CRS before declaring georeferencing correct.
5. **NEVER** assume georeferencing exists — explicitly check for `IfcMapConversion` and `IfcProjectedCRS`.
6. **NEVER** mix coordinates from different UTM zones in a single project.
7. **NEVER** assume heights are ellipsoidal — BIM models use orthometric heights by default.
8. **NEVER** skip the direction vector normalization check for True North rotation.
9. **ALWAYS** check for compound errors — multiple failure modes can occur simultaneously.
10. **ALWAYS** verify fixes with a known reference point before declaring the issue resolved.

---

## Reference Links

- [references/methods.md](references/methods.md) — Diagnostic API signatures for coordinate issues per tool
- [references/examples.md](references/examples.md) — Real coordinate error scenarios with step-by-step fixes
- [references/anti-patterns.md](references/anti-patterns.md) — Coordinate handling mistakes to avoid

### Official Sources

- https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/schema/ifcrepresentationresource/lexical/ifcmapconversion.htm
- https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/schema/ifcrepresentationresource/lexical/ifcprojectedcrs.htm
- https://pyproj4.github.io/pyproj/stable/api/transformer.html
- https://ifcopenshell.github.io/docs/python/html/ifcopenshell-python/geometry_processing.html
- https://epsg.io/
