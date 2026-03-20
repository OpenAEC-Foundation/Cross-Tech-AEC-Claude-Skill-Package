# examples.md — Working Transformation Code Examples

## Example 1: Complete BIM → GIS Pipeline (IfcOpenShell + pyproj)

```python
import ifcopenshell
import ifcopenshell.util.geolocation as geo
from pyproj import Transformer

# Step 1: Open IFC and verify georeferencing
ifc = ifcopenshell.open("building.ifc")

map_conversions = ifc.by_type("IfcMapConversion")
projected_crs_list = ifc.by_type("IfcProjectedCRS")

if not map_conversions or not projected_crs_list:
    raise ValueError("Model lacks georeferencing — cannot place on map")

# Step 2: Read CRS info
crs_name = projected_crs_list[0].Name  # e.g., "EPSG:28992"

# Step 3: Transform local coordinates to map CRS
local_x, local_y, local_z = 10.0, 20.0, 0.0
e, n, h = geo.auto_xyz2enh(ifc, local_x, local_y, local_z)
# Result: map coordinates in the CRS defined by IfcProjectedCRS

# Step 4: Optional — reproject to WGS84 for web maps
transformer = Transformer.from_crs(crs_name, "EPSG:4326", always_xy=True)
lon, lat = transformer.transform(e, n)
```

## Example 2: Add Georeferencing to an Ungeoreferenced IFC File

```python
import ifcopenshell
import ifcopenshell.api

ifc = ifcopenshell.open("model_without_georef.ifc")

# Add empty georeferencing entities
ifcopenshell.api.run("georeference.add_georeferencing", ifc)

# Configure for Dutch project (RD New)
ifcopenshell.api.run("georeference.edit_georeferencing", ifc,
    projected_crs={"Name": "EPSG:28992"},
    coordinate_operation={
        "Eastings": 121687.0,      # Amsterdam Centraal approximate easting
        "Northings": 487316.0,     # Amsterdam Centraal approximate northing
        "OrthogonalHeight": -2.0,  # Below NAP (Amsterdam is below sea level)
        "XAxisAbscissa": 1.0,      # No rotation (project north = true north)
        "XAxisOrdinate": 0.0,
        "Scale": 1.0               # Model units = meters = CRS units
    }
)

ifc.write("model_georeferenced.ifc")
```

## Example 3: Add Georeferencing with True North Rotation

```python
import math
import ifcopenshell
import ifcopenshell.api

ifc = ifcopenshell.open("model.ifc")

# True north is 30° anti-clockwise from project north
angle_deg = 30.0
angle_rad = math.radians(angle_deg)

ifcopenshell.api.run("georeference.add_georeferencing", ifc)
ifcopenshell.api.run("georeference.edit_georeferencing", ifc,
    projected_crs={"Name": "EPSG:28992"},
    coordinate_operation={
        "Eastings": 155000.0,
        "Northings": 463000.0,
        "OrthogonalHeight": 0.0,
        "XAxisAbscissa": math.cos(angle_rad),  # 0.866025
        "XAxisOrdinate": math.sin(angle_rad),   # 0.5
        "Scale": 1.0
    }
)

ifc.write("model_rotated.ifc")
```

## Example 4: Manual IfcMapConversion Transformation (without IfcOpenShell utility)

```python
import math

def ifc_local_to_map(x, y, z, eastings, northings, ortho_height,
                     x_axis_abscissa=1.0, x_axis_ordinate=0.0, scale=1.0):
    """Transform local BIM coordinates to map CRS coordinates."""
    e = scale * (x * x_axis_abscissa - y * x_axis_ordinate) + eastings
    n = scale * (x * x_axis_ordinate + y * x_axis_abscissa) + northings
    h = scale * z + ortho_height
    return e, n, h

def ifc_map_to_local(e, n, h, eastings, northings, ortho_height,
                     x_axis_abscissa=1.0, x_axis_ordinate=0.0, scale=1.0):
    """Transform map CRS coordinates to local BIM coordinates."""
    dx = e - eastings
    dy = n - northings
    x_scaled = dx * x_axis_abscissa + dy * x_axis_ordinate
    y_scaled = dy * x_axis_abscissa - dx * x_axis_ordinate
    x = x_scaled / scale
    y = y_scaled / scale
    z = (h - ortho_height) / scale
    return x, y, z

# Usage
e, n, h = ifc_local_to_map(
    x=10.0, y=20.0, z=3.0,
    eastings=155000.0, northings=463000.0, ortho_height=0.0,
    x_axis_abscissa=1.0, x_axis_ordinate=0.0, scale=1.0
)
# Result: (155010.0, 463020.0, 3.0)
```

## Example 5: pyproj Batch Transformation with NumPy

```python
import numpy as np
from pyproj import Transformer

# WGS84 → RD New for multiple points
transformer = Transformer.from_crs("EPSG:4326", "EPSG:28992", always_xy=True)

lons = np.array([4.9003, 5.1214, 4.3571, 5.4697])
lats = np.array([52.3792, 52.0907, 52.0116, 51.4416])

eastings, northings = transformer.transform(lons, lats)
# eastings: array of RD New easting values
# northings: array of RD New northing values
```

## Example 6: 3D Transformation with Vertical Datum (Netherlands)

```python
from pyproj import Transformer

# RD New + NAP → WGS84 3D (with vertical datum conversion)
transformer_3d = Transformer.from_crs(
    "EPSG:7415",   # Amersfoort/RD New + NAP (compound CRS)
    "EPSG:4979",   # WGS84 geographic 3D (lon, lat, ellipsoidal height)
    always_xy=True
)

# NAP height = 10.0m
lon, lat, ellipsoidal_h = transformer_3d.transform(155000, 463000, 10.0)
# ellipsoidal_h ≈ 52–54m (NAP + ~42m geoid undulation in NL)
```

## Example 7: IFC to QGIS via GeoPackage

```python
import ifcopenshell
import ifcopenshell.util.geolocation as geo
import geopandas as gpd
from shapely.geometry import Point

# Open IFC and verify georeferencing
ifc = ifcopenshell.open("building.ifc")
if not ifc.by_type("IfcMapConversion"):
    raise ValueError("No georeferencing in IFC file")

crs_name = ifc.by_type("IfcProjectedCRS")[0].Name  # "EPSG:28992"

# Extract wall locations and transform
walls = ifc.by_type("IfcWall")
records = []
for wall in walls:
    # Simplified: extract centroid from placement
    local_x, local_y, local_z = 0.0, 0.0, 0.0  # Extract from wall.ObjectPlacement
    e, n, h = geo.auto_xyz2enh(ifc, local_x, local_y, local_z)
    records.append({
        "GlobalId": wall.GlobalId,
        "Name": wall.Name or "",
        "Height": h,
        "geometry": Point(e, n)
    })

# Create GeoDataFrame and export
gdf = gpd.GeoDataFrame(records, crs=crs_name)
gdf.to_file("building_walls.gpkg", driver="GPKG")
# Open in QGIS: Layer → Add Layer → Add Vector Layer → building_walls.gpkg
```

## Example 8: BIM Z-up to Three.js Y-up Conversion

```javascript
// JavaScript: IFC Z-up → Three.js Y-up
function bimToThreeJs(x, y, z) {
    return {
        x: x,       // BIM X → Three.js X (east)
        y: z,        // BIM Z (up) → Three.js Y (up)
        z: -y        // BIM Y (north) → Three.js -Z (forward)
    };
}

// Inverse: Three.js Y-up → BIM Z-up
function threeJsToBim(x, y, z) {
    return {
        x: x,       // Three.js X → BIM X
        y: -z,       // Three.js -Z → BIM Y (north)
        z: y         // Three.js Y (up) → BIM Z (up)
    };
}

// Usage with Three.js mesh
const bimCoord = { x: 10, y: 20, z: 3 };
const threeCoord = bimToThreeJs(bimCoord.x, bimCoord.y, bimCoord.z);
mesh.position.set(threeCoord.x, threeCoord.y, threeCoord.z);
```

## Example 9: Validate Coordinates Against CRS Range

```python
CRS_RANGES = {
    "EPSG:28992": {"e_min": -7000, "e_max": 300000, "n_min": 289000, "n_max": 629000},
    "EPSG:32631": {"e_min": 166000, "e_max": 834000, "n_min": 0, "n_max": 9400000},
    "EPSG:32632": {"e_min": 166000, "e_max": 834000, "n_min": 0, "n_max": 9400000},
}

def validate_coordinates(easting, northing, epsg_code):
    """Check if coordinates fall within the valid range for a CRS."""
    key = f"EPSG:{epsg_code}" if isinstance(epsg_code, int) else epsg_code
    if key not in CRS_RANGES:
        return True  # Unknown CRS — cannot validate
    r = CRS_RANGES[key]
    if not (r["e_min"] <= easting <= r["e_max"]):
        raise ValueError(f"Easting {easting} outside valid range for {key}")
    if not (r["n_min"] <= northing <= r["n_max"]):
        raise ValueError(f"Northing {northing} outside valid range for {key}")
    return True

# Usage
validate_coordinates(155000, 463000, "EPSG:28992")  # OK
validate_coordinates(155000, 463000, "EPSG:32631")  # Raises — values match RD, not UTM
```

## Example 10: Extract IfcMapConversion Parameters Manually

```python
import ifcopenshell

ifc = ifcopenshell.open("model.ifc")

mc = ifc.by_type("IfcMapConversion")
if not mc:
    raise ValueError("No IfcMapConversion found")

mc = mc[0]
params = {
    "Eastings": mc.Eastings,
    "Northings": mc.Northings,
    "OrthogonalHeight": mc.OrthogonalHeight,
    "XAxisAbscissa": mc.XAxisAbscissa if mc.XAxisAbscissa is not None else 1.0,
    "XAxisOrdinate": mc.XAxisOrdinate if mc.XAxisOrdinate is not None else 0.0,
    "Scale": mc.Scale if mc.Scale is not None else 1.0,
}

crs = ifc.by_type("IfcProjectedCRS")
if crs:
    params["CRS_Name"] = crs[0].Name
    params["GeodeticDatum"] = crs[0].GeodeticDatum
    params["VerticalDatum"] = crs[0].VerticalDatum

print(params)
```
