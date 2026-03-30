---
name: crosstech-core-coordinate-systems
description: >
  Use when transforming coordinates between BIM local systems and GIS global systems,
  handling CRS conversions, or debugging models appearing at wrong locations.
  Prevents the common mistake of applying wrong EPSG codes, ignoring axis conventions
  (Y-up vs Z-up), or losing precision in coordinate transformations.
  Covers IfcMapConversion, IfcProjectedCRS, EPSG codes, pyproj/PROJ transformations,
  and axis conventions for Blender, Three.js, QGIS, Revit, FreeCAD, and IFC.
  Keywords: CRS, EPSG, coordinate system, georeferencing, IfcMapConversion,
  IfcProjectedCRS, pyproj, PROJ, Y-up, Z-up, BIM to GIS, map conversion,
  model at wrong location, coordinate transform, which EPSG code.
license: MIT
compatibility: "Designed for Claude Code. Covers IFC4/IFC4X3 georeferencing, PROJ 9.x, pyproj 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-core-coordinate-systems

## Quick Reference

### Key EPSG Codes for AEC

| EPSG | Name | Type | Units | Use Case |
|------|------|------|-------|----------|
| **4326** | WGS 84 | Geographic 2D | Degrees | GPS, global web maps, Speckle default |
| **28992** | Amersfoort / RD New | Projected | Meters | Netherlands cadastre, engineering |
| **7415** | RD New + NAP height | Compound | Meters | Netherlands 3D (horizontal + vertical) |
| **32631** | WGS 84 / UTM 31N | Projected | Meters | Western Netherlands, Belgium |
| **32632** | WGS 84 / UTM 32N | Projected | Meters | Eastern Netherlands, Germany |
| **3857** | Pseudo-Mercator | Projected | Meters | Web maps (OSM, Google Maps) |
| **5709** | NAP height | Vertical | Meters | Netherlands vertical datum |

### Axis Conventions by Tool

| Tool | Up Axis | Handedness | Default Units | Notes |
|------|---------|-----------|---------------|-------|
| **Blender** | Z-up | Right | Meters | Bonsai aligns with IFC |
| **Three.js** | Y-up | Right | Unitless | ALWAYS swap Y/Z from BIM |
| **QGIS** | Z-up (2.5D) | Right | Map units | Z for 2.5D features only |
| **IFC** | Z-up | Right | Project units | mm or m via IfcUnitAssignment |
| **Revit** | Z-up | Right | Internal: feet | API returns feet |
| **FreeCAD** | Z-up | Right | Internal: mm | Arch workbench: mm |
| **Speckle** | Z-up | Right | Meters | Converts per target app |
| **web-ifc** | Z-up | Right | IFC project units | Same as source IFC |

### Critical Warnings

**NEVER** use pyproj without `always_xy=True` — EPSG:4326 defaults to (lat, lon) order, causing silent coordinate swaps.

**NEVER** import IFC into GIS without checking for IfcMapConversion — models without georeferencing land at (0°N, 0°E) in the Gulf of Guinea.

**NEVER** mix UTM zones in a single project — coordinates from adjacent zones differ by hundreds of kilometers in easting.

**NEVER** confuse NAP height with WGS84 ellipsoidal height — the ~42m difference in the Netherlands causes buildings to float above terrain.

**NEVER** hardcode TOWGS84 parameters — use EPSG codes via pyproj, which selects the best available transformation automatically.

**ALWAYS** apply Y/Z swap when moving data between Z-up (BIM/GIS) and Y-up (Three.js) environments.

**ALWAYS** verify georeferencing by overlaying the result on a known map — visual checks catch errors that math alone misses.

---

## Technology Boundary

### Side A: BIM Local Coordinates

BIM authoring tools use local coordinate systems with no inherent geographic meaning. Geometry sits at or near local origin (0,0,0).

**Key concepts:**
- **Revit Internal Origin**: Fixed at (0,0,0), immutable. Survey Point defines real-world relationship. Internal units: feet (1 ft = 0.3048 m exactly).
- **Blender World Origin**: (0,0,0). Default units: meters. Bonsai handles georeferencing via IFC entities.
- **FreeCAD World Origin**: (0,0,0). Internal units: millimeters.
- **Typical coordinate range**: 0 to 500,000 mm (large building ~500m span).

**IFC project units** are defined in `IfcProject` → `IfcUnitAssignment`. Common values: meters (m) or millimeters (mm). The `IfcMapConversion.Scale` factor converts from project units to CRS map units.

### Side B: GIS Global Coordinates (CRS-based)

GIS tools use Coordinate Reference Systems (CRS) to map coordinates to real-world locations on Earth.

**Two CRS types:**
- **Geographic CRS**: Latitude/longitude in degrees on an ellipsoid (e.g., EPSG:4326 WGS84). No distortion but not usable for engineering measurements.
- **Projected CRS**: Easting/northing in meters on a flat plane (e.g., EPSG:28992 RD New). ALWAYS introduces some distortion. Required for engineering work.

**Datum transformations** between different ellipsoids are ALWAYS lossy:
- 7-parameter Helmert: 1–5 m accuracy
- Grid-based (NTv2/GTX): 1–10 cm accuracy
- Same-datum reprojections: lossless (float precision only)

### The Bridge: IfcMapConversion + pyproj Transformation

The bridge between BIM local and GIS global has two stages:

**Stage 1 — IfcMapConversion** (local BIM → projected CRS):

```
E = Scale * (x * XAxisAbscissa - y * XAxisOrdinate) + Eastings
N = Scale * (x * XAxisOrdinate + y * XAxisAbscissa) + Northings
H = Scale * z + OrthogonalHeight
```

Where:
- `(x, y, z)` = local BIM coordinates
- `(E, N, H)` = map coordinates in the target CRS
- `XAxisAbscissa = cos(θ)`, `XAxisOrdinate = sin(θ)` where θ = true north rotation
- `Scale` converts project units to CRS units (e.g., 0.001 for mm→m)

**Stage 2 — pyproj** (projected CRS → any other CRS):

```python
from pyproj import Transformer
transformer = Transformer.from_crs("EPSG:28992", "EPSG:4326", always_xy=True)
lon, lat = transformer.transform(easting, northing)
```

**IFC entities involved:**

| Entity | Purpose | Key Attributes |
|--------|---------|----------------|
| `IfcProjectedCRS` | Defines target CRS | `Name` (EPSG code), `MapUnit` |
| `IfcMapConversion` | Defines transformation | `Eastings`, `Northings`, `OrthogonalHeight`, `XAxisAbscissa`, `XAxisOrdinate`, `Scale` |

**IFC4 vs IFC4X3 differences:**

| Feature | IFC4 | IFC4X3 |
|---------|------|--------|
| IfcMapConversion | Available | Retained |
| IfcMapConversionScaled | NOT available | NEW — per-axis scale factors |
| IfcRigidOperation | NOT available | NEW — no-scaling alternative |
| IfcProjectedCRS.GeodeticDatum | Mandatory | Changed to OPTIONAL |

---

## Critical Rules (ALWAYS/NEVER)

1. **ALWAYS** set `always_xy=True` in pyproj `Transformer.from_crs()` calls.
2. **ALWAYS** check for both `IfcMapConversion` AND `IfcProjectedCRS` before BIM→GIS conversion.
3. **ALWAYS** verify that `IfcMapConversion.Scale` matches the ratio between project units and CRS units.
4. **ALWAYS** use `IfcProjectedCRS.Name` in the format `"EPSG:XXXXX"` for unambiguous CRS identification.
5. **ALWAYS** validate coordinates fall within the valid range for the assigned CRS after transformation.
6. **ALWAYS** use `ifcopenshell.util.geolocation.auto_xyz2enh` when the IFC file contains georeferencing — it reads parameters directly from the model.
7. **NEVER** assume model units — read them from `IfcProject` → `IfcUnitAssignment`.
8. **NEVER** use EPSG:3857 (Web Mercator) for engineering work — it distorts areas and distances significantly.
9. **NEVER** omit the Y/Z swap when transferring between Z-up and Y-up systems.
10. **NEVER** mix vertical datums (NAP vs ellipsoidal) in a single pipeline without explicit conversion.

---

## Decision Tree

```
Is this a BIM → GIS conversion?
├─ YES → Does the IFC file contain IfcMapConversion?
│   ├─ YES → Use auto_xyz2enh() for local → projected CRS
│   │   └─ Need a different target CRS?
│   │       ├─ YES → Chain with pyproj Transformer
│   │       └─ NO → Done (coordinates are in IfcProjectedCRS target)
│   └─ NO → STOP. Georeferencing MUST be added first.
│       └─ Use ifcopenshell.api.run("georeference.add_georeferencing")
│
Is this a GIS → BIM conversion?
├─ YES → Use pyproj to get coordinates in the IFC's target CRS
│   └─ Then use enh2xyz() to convert to local BIM coordinates
│
Is this a BIM → Web viewer (Three.js) conversion?
├─ YES → Apply Y/Z swap: Three.js_y = BIM_z, Three.js_z = -BIM_y
│   └─ Georeferencing needed? → First do BIM → GIS, then GIS → viewer
│
Is the model appearing at the wrong location?
├─ YES → See "Common Failures" in references/anti-patterns.md
│   ├─ At (0,0) / Gulf of Guinea? → Missing IfcMapConversion
│   ├─ Hundreds of km off? → Wrong EPSG code or UTM zone mismatch
│   ├─ ~42m vertical offset (NL)? → NAP vs ellipsoidal height confusion
│   ├─ 1000x too large/small? → Unit mismatch (mm vs m)
│   └─ Rotated on map? → Wrong XAxisAbscissa/XAxisOrdinate values
```

---

## Essential Patterns

### Pattern 1: Check for Georeferencing

```python
import ifcopenshell

ifc = ifcopenshell.open("model.ifc")
has_georef = len(ifc.by_type("IfcMapConversion")) > 0
has_crs = len(ifc.by_type("IfcProjectedCRS")) > 0

if not has_georef or not has_crs:
    raise ValueError("Model lacks georeferencing — cannot place on map")
```

### Pattern 2: Local to Map Coordinates (IfcOpenShell)

```python
import ifcopenshell.util.geolocation as geo

# Auto-reads IfcMapConversion from the file
e, n, h = geo.auto_xyz2enh(ifc, local_x, local_y, local_z)
```

### Pattern 3: CRS Reprojection (pyproj)

```python
from pyproj import Transformer

transformer = Transformer.from_crs("EPSG:28992", "EPSG:4326", always_xy=True)
lon, lat = transformer.transform(easting, northing)
```

### Pattern 4: BIM Z-up to Three.js Y-up

```javascript
function bimToThreeJs(x, y, z) {
    return { x: x, y: z, z: -y };
}
```

The negation of Y→Z preserves the right-handed coordinate system.

### Pattern 5: Add Georeferencing to IFC

```python
import ifcopenshell
import ifcopenshell.api

ifc = ifcopenshell.open("model.ifc")
ifcopenshell.api.run("georeference.add_georeferencing", ifc)
ifcopenshell.api.run("georeference.edit_georeferencing", ifc,
    projected_crs={"Name": "EPSG:28992"},
    coordinate_operation={
        "Eastings": 155000.0,
        "Northings": 463000.0,
        "OrthogonalHeight": 0.0,
        "XAxisAbscissa": 1.0,
        "XAxisOrdinate": 0.0,
        "Scale": 1.0
    }
)
ifc.write("georeferenced_model.ifc")
```

---

## Common Operations

### Validate CRS Coordinate Ranges

| CRS | Valid Easting Range | Valid Northing Range |
|-----|-------------------|---------------------|
| EPSG:28992 (RD New) | -7,000 to 300,000 | 289,000 to 629,000 |
| EPSG:32631 (UTM 31N) | 166,000 to 834,000 | 0 to 9,400,000 |
| EPSG:32632 (UTM 32N) | 166,000 to 834,000 | 0 to 9,400,000 |

### True North Rotation Encoding

For a θ-degree anti-clockwise True North rotation:
```
XAxisAbscissa = cos(θ)
XAxisOrdinate = sin(θ)
```

Example: 30° rotation → XAxisAbscissa = 0.866025, XAxisOrdinate = 0.5

### Vertical Datum: NAP vs Ellipsoidal

In the Netherlands, NAP heights differ from WGS84 ellipsoidal heights by ~42–44 meters (geoid undulation). For 3D transformations, use compound CRS EPSG:7415 (RD New + NAP) with pyproj to handle vertical conversion automatically.

---

## Reference Links

- [references/methods.md](references/methods.md) — CRS transformation APIs: pyproj, IfcOpenShell georef, PROJ pipeline syntax
- [references/examples.md](references/examples.md) — Working transformation code examples for all patterns
- [references/anti-patterns.md](references/anti-patterns.md) — Coordinate system mistakes and how to diagnose them

### Official Sources

- https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/ (IFC4 specification)
- https://ifc43-docs.standards.buildingsmart.org/ (IFC4X3 specification)
- https://proj.org/en/stable/ (PROJ documentation)
- https://pyproj4.github.io/pyproj/stable/ (pyproj documentation)
- https://docs.ifcopenshell.org/ (IfcOpenShell documentation)
- https://epsg.io (EPSG registry lookup)
