# methods.md — Diagnostic APIs for Coordinate Issues

## IfcOpenShell Georeferencing Detection

### Check Georeferencing Presence

```python
import ifcopenshell

def diagnose_georeferencing(filepath: str) -> dict:
    """Return a diagnostic report for georeferencing in an IFC file.

    ALWAYS call this FIRST when debugging any coordinate issue.
    """
    ifc = ifcopenshell.open(filepath)
    report = {
        "has_map_conversion": False,
        "has_projected_crs": False,
        "crs_name": None,
        "eastings": None,
        "northings": None,
        "orthogonal_height": None,
        "scale": None,
        "x_axis_abscissa": None,
        "x_axis_ordinate": None,
        "vertical_datum": None,
        "project_length_unit": None,
    }

    # Check IfcMapConversion
    map_convs = ifc.by_type("IfcMapConversion")
    if map_convs:
        mc = map_convs[0]
        report["has_map_conversion"] = True
        report["eastings"] = mc.Eastings
        report["northings"] = mc.Northings
        report["orthogonal_height"] = mc.OrthogonalHeight
        report["scale"] = mc.Scale  # None means 1.0
        report["x_axis_abscissa"] = mc.XAxisAbscissa  # None means 1.0
        report["x_axis_ordinate"] = mc.XAxisOrdinate  # None means 0.0

    # Check IfcProjectedCRS
    crs_list = ifc.by_type("IfcProjectedCRS")
    if crs_list:
        crs = crs_list[0]
        report["has_projected_crs"] = True
        report["crs_name"] = crs.Name
        report["vertical_datum"] = crs.VerticalDatum

    # Check project length units
    for ua in ifc.by_type("IfcUnitAssignment"):
        for unit in ua.Units:
            if hasattr(unit, "UnitType") and unit.UnitType == "LENGTHUNIT":
                if hasattr(unit, "Prefix") and unit.Prefix == "MILLI":
                    report["project_length_unit"] = "MILLIMETRE"
                elif hasattr(unit, "Name"):
                    report["project_length_unit"] = unit.Name

    return report
```

### Read True North Angle

```python
import math
import ifcopenshell

def get_true_north_angle(ifc) -> float | None:
    """Return True North angle in degrees (anti-clockwise from Y-axis).

    Returns None if no rotation is defined.
    """
    map_convs = ifc.by_type("IfcMapConversion")
    if map_convs:
        xa = map_convs[0].XAxisAbscissa
        xo = map_convs[0].XAxisOrdinate
        if xa is not None and xo is not None:
            return math.degrees(math.atan2(xo, xa))

    # Fallback: check IfcGeometricRepresentationContext.TrueNorth
    for ctx in ifc.by_type("IfcGeometricRepresentationContext"):
        if ctx.TrueNorth:
            direction = ctx.TrueNorth.DirectionRatios
            return math.degrees(math.atan2(direction[0], direction[1]))

    return None
```

---

## IfcOpenShell Coordinate Transformation

### Local to Map Coordinates

```python
import ifcopenshell.util.geolocation as geo

def local_to_map(ifc, x: float, y: float, z: float) -> tuple:
    """Transform local BIM coordinates to map coordinates.

    ALWAYS use auto_xyz2enh — it reads IfcMapConversion automatically.
    """
    return geo.auto_xyz2enh(ifc, x, y, z)


def map_to_local(ifc, e: float, n: float, h: float) -> tuple:
    """Transform map coordinates to local BIM coordinates."""
    return geo.auto_enh2xyz(ifc, e, n, h)
```

### Manual Transformation (when auto functions are unavailable)

```python
import ifcopenshell.util.geolocation as geo

def manual_local_to_map(x, y, z, eastings, northings, orth_height,
                        xa=1.0, xo=0.0, scale=1.0):
    """Manual transformation using xyz2enh.

    Use when IfcMapConversion parameters are known but not in the IFC file.
    """
    return geo.xyz2enh(
        x, y, z,
        eastings=eastings,
        northings=northings,
        orthogonal_height=orth_height,
        x_axis_abscissa=xa,
        x_axis_ordinate=xo,
        scale=scale,
        factor_x=1.0, factor_y=1.0, factor_z=1.0  # IFC4 defaults
    )
```

---

## pyproj CRS Validation

### Validate Coordinates Against CRS Bounds

```python
from pyproj import CRS

def validate_coordinates_in_crs(epsg_code: int, easting: float, northing: float) -> bool:
    """Check if coordinates fall within the valid area of use for the CRS.

    ALWAYS call this after reading IfcMapConversion Eastings/Northings.
    Returns True if coordinates are within bounds.
    """
    crs = CRS.from_epsg(epsg_code)
    bounds = crs.area_of_use.bounds  # (west_lon, south_lat, east_lon, north_lat)

    # For projected CRS, transform bounds to projected coordinates
    from pyproj import Transformer
    t = Transformer.from_crs("EPSG:4326", f"EPSG:{epsg_code}", always_xy=True)

    min_e, min_n = t.transform(bounds[0], bounds[1])
    max_e, max_n = t.transform(bounds[2], bounds[3])

    in_range = (min_e <= easting <= max_e) and (min_n <= northing <= max_n)
    if not in_range:
        print(f"OUT OF RANGE: ({easting}, {northing}) not in "
              f"EPSG:{epsg_code} bounds ({min_e:.0f}-{max_e:.0f}, {min_n:.0f}-{max_n:.0f})")
    return in_range
```

### CRS Comparison

```python
from pyproj import CRS

def compare_crs(epsg_a: int, epsg_b: int) -> dict:
    """Compare two CRS definitions to understand their relationship."""
    crs_a = CRS.from_epsg(epsg_a)
    crs_b = CRS.from_epsg(epsg_b)
    return {
        "same_datum": crs_a.datum == crs_b.datum,
        "both_projected": crs_a.is_projected and crs_b.is_projected,
        "crs_a_units": crs_a.axis_info[0].unit_name if crs_a.axis_info else None,
        "crs_b_units": crs_b.axis_info[0].unit_name if crs_b.axis_info else None,
        "transformation_lossy": crs_a.datum != crs_b.datum,
    }
```

---

## Three.js Axis Swap Detection

### Bounding Box Analysis

```javascript
// Three.js: detect Y/Z axis swap from loaded BIM model
function detectAxisSwap(object3D) {
    const box = new THREE.Box3().setFromObject(object3D);
    const size = box.getSize(new THREE.Vector3());

    // BIM buildings: height (Z in BIM) should map to Y in Three.js
    // If Y dimension is near zero but Z is large → swap was not applied
    if (size.y < 0.5 && size.z > 2.0) {
        return {
            swapNeeded: true,
            reason: "Model appears flat in Y — likely Z-up data loaded without Y/Z swap"
        };
    }
    return { swapNeeded: false };
}
```

### Apply Axis Correction

```javascript
// Three.js: rotate entire scene to convert Z-up → Y-up
function applyZUpToYUp(object3D) {
    // Rotate -90° around X axis: Z-up becomes Y-up
    object3D.rotation.x = -Math.PI / 2;
    object3D.updateMatrixWorld(true);
}
```

---

## QGIS Layer CRS Verification

### Python Console (QGIS)

```python
# QGIS Python Console: check CRS of active layer
layer = iface.activeLayer()
crs = layer.crs()
print(f"Layer CRS: {crs.authid()}")       # e.g., "EPSG:28992"
print(f"Units: {crs.mapUnits()}")          # e.g., QgsUnitTypes.DistanceMeters
print(f"Is geographic: {crs.isGeographic()}")
print(f"Description: {crs.description()}")

# Check project CRS
project_crs = QgsProject.instance().crs()
if layer.crs() != project_crs:
    print(f"WARNING: Layer CRS ({crs.authid()}) differs from "
          f"project CRS ({project_crs.authid()}) — on-the-fly reprojection active")
```

---

## Blender / Bonsai Georeferencing Check

### Blender Python Console

```python
import bpy
import ifcopenshell

# Get the active IFC file from Bonsai
ifc_file = ifcopenshell.open(bpy.context.scene.BIMProperties.ifc_file)

# Check georeferencing
map_convs = ifc_file.by_type("IfcMapConversion")
if map_convs:
    mc = map_convs[0]
    print(f"Eastings: {mc.Eastings}")
    print(f"Northings: {mc.Northings}")
    print(f"Height: {mc.OrthogonalHeight}")
    print(f"Scale: {mc.Scale}")
else:
    print("NO GEOREFERENCING — model has no map conversion")
```

---

## Revit Coordinate Extraction

### Revit API (C#)

```csharp
// Revit API: extract survey point and project base point
ProjectLocation projLoc = doc.ActiveProjectLocation;
ProjectPosition pos = projLoc.GetProjectPosition(XYZ.Zero);

// pos.EastWest and pos.NorthSouth are in FEET
double eastingFeet = pos.EastWest;
double northingFeet = pos.NorthSouth;
double elevationFeet = pos.Elevation;
double angleRad = pos.Angle;  // True North rotation in radians

// Convert to meters
double eastingM = eastingFeet * 0.3048;
double northingM = northingFeet * 0.3048;
double elevationM = elevationFeet * 0.3048;

// CRITICAL: Revit API ALWAYS returns feet, regardless of display units
```

### Revit Dynamo (Python)

```python
# Dynamo Python Script node
import clr
clr.AddReference("RevitAPI")
from Autodesk.Revit.DB import *

doc = __revit__.ActiveUIDocument.Document
proj_loc = doc.ActiveProjectLocation
pos = proj_loc.GetProjectPosition(XYZ.Zero)

# Values in feet — ALWAYS multiply by 0.3048 for meters
OUT = {
    "easting_ft": pos.EastWest,
    "northing_ft": pos.NorthSouth,
    "elevation_ft": pos.Elevation,
    "true_north_rad": pos.Angle
}
```
