# crosstech-impl-qgis-bim-georef — Examples

## Example 1: Complete IFC-to-QGIS Pipeline via GeoPackage

End-to-end pipeline: extract wall footprints from a georeferenced IFC file, export as GeoPackage, and load into QGIS with an OpenStreetMap basemap.

```python
import ifcopenshell
import ifcopenshell.geom
import ifcopenshell.util.geolocation as geo
import multiprocessing
from shapely.geometry import MultiPoint
import geopandas as gpd

# --- Step 1: Open IFC and verify georeferencing ---
ifc = ifcopenshell.open("office_building.ifc")

map_conversions = ifc.by_type("IfcMapConversion")
projected_crs_list = ifc.by_type("IfcProjectedCRS")

if not map_conversions or not projected_crs_list:
    raise ValueError("IFC file lacks georeferencing — add IfcMapConversion first")

epsg_code = projected_crs_list[0].Name  # e.g., "EPSG:28992"

# --- Step 2: Extract footprints with geometry iterator ---
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)
settings.set(settings.WELD_VERTICES, True)

iterator = ifcopenshell.geom.iterator(
    settings, ifc, multiprocessing.cpu_count()
)

features = []
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = ifc.by_id(shape.id)

        # Only process building elements (walls, slabs, columns)
        if not element.is_a("IfcBuildingElement"):
            if not iterator.next():
                break
            continue

        verts = shape.geometry.verts
        coords_2d = [(verts[i], verts[i + 1])
                      for i in range(0, len(verts), 3)]

        if len(coords_2d) >= 3:
            footprint = MultiPoint(coords_2d).convex_hull
            if not footprint.is_empty and footprint.area > 0.01:
                features.append({
                    "geometry": footprint,
                    "GlobalId": element.GlobalId,
                    "IfcType": element.is_a(),
                    "Name": getattr(element, "Name", None) or "",
                })

        if not iterator.next():
            break

# --- Step 3: Transform to map coordinates and export ---
# Note: USE_WORLD_COORDS gives local world coords.
# For georeferenced coords, use auto_xyz2enh on centroids
# or ensure the IFC local coords already include the offset.

gdf = gpd.GeoDataFrame(features, crs=epsg_code)
gdf.to_file("office_footprints.gpkg", driver="GPKG")
print(f"Exported {len(features)} footprints to GeoPackage")

# --- Step 4: Load into QGIS ---
from qgis.core import (
    QgsVectorLayer, QgsRasterLayer, QgsProject
)

# Load footprints
footprint_layer = QgsVectorLayer(
    "office_footprints.gpkg", "BIM Footprints", "ogr"
)
if footprint_layer.isValid():
    QgsProject.instance().addMapLayer(footprint_layer)

# Add OpenStreetMap basemap
osm_url = (
    "type=xyz&url=https://tile.openstreetmap.org/"
    "{z}/{x}/{y}.png&zmax=19&zmin=0"
)
basemap = QgsRasterLayer(osm_url, "OpenStreetMap", "wms")
if basemap.isValid():
    QgsProject.instance().addMapLayer(basemap, False)
    root = QgsProject.instance().layerTreeRoot()
    root.insertLayer(-1, basemap)
```

---

## Example 2: Per-Storey Footprint Export

Filter IFC elements by building storey to avoid overlapping 2D footprints.

```python
import ifcopenshell
import ifcopenshell.geom
import ifcopenshell.util.geolocation as geo
from shapely.geometry import MultiPoint
import geopandas as gpd

ifc = ifcopenshell.open("multi_storey.ifc")
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

# Collect elements per storey
storey_elements = {}
for storey in ifc.by_type("IfcBuildingStorey"):
    elements = []
    for rel in storey.ContainsElements:
        elements.extend(rel.RelatedElements)
    storey_elements[storey.Name or storey.GlobalId] = elements

# Extract footprints per storey
for storey_name, elements in storey_elements.items():
    features = []
    for element in elements:
        try:
            shape = ifcopenshell.geom.create_shape(settings, element)
            verts = shape.geometry.verts
            coords_2d = [(verts[i], verts[i + 1])
                          for i in range(0, len(verts), 3)]

            if len(coords_2d) >= 3:
                footprint = MultiPoint(coords_2d).convex_hull
                if not footprint.is_empty and footprint.area > 0.01:
                    features.append({
                        "geometry": footprint,
                        "GlobalId": element.GlobalId,
                        "IfcType": element.is_a(),
                        "Name": getattr(element, "Name", None) or "",
                        "Storey": storey_name,
                    })
        except Exception:
            continue

    if features:
        gdf = gpd.GeoDataFrame(features, crs="EPSG:28992")
        safe_name = storey_name.replace(" ", "_").replace("/", "_")
        gdf.to_file(f"footprints_{safe_name}.gpkg", driver="GPKG")
        print(f"Storey '{storey_name}': {len(features)} elements exported")
```

---

## Example 3: IFC Point Cloud to QGIS with CRS Reprojection

Extract element centroid locations, transform from RD New to WGS84, and create a QGIS layer.

```python
import ifcopenshell
import ifcopenshell.util.geolocation as geo
from pyproj import Transformer
from qgis.core import (
    QgsVectorLayer, QgsProject, QgsFeature,
    QgsGeometry, QgsPointXY, QgsField
)
from PyQt5.QtCore import QVariant

ifc = ifcopenshell.open("building.ifc")

# Verify georeferencing exists
if not ifc.by_type("IfcMapConversion"):
    raise ValueError("No IfcMapConversion found")

# Set up CRS reprojection: RD New -> WGS84
transformer = Transformer.from_crs("EPSG:28992", "EPSG:4326", always_xy=True)

# Create WGS84 point layer
layer = QgsVectorLayer("Point?crs=EPSG:4326", "BIM Locations (WGS84)", "memory")
provider = layer.dataProvider()
provider.addAttributes([
    QgsField("GlobalId", QVariant.String),
    QgsField("Name", QVariant.String),
    QgsField("IfcType", QVariant.String),
    QgsField("Easting_RD", QVariant.Double),
    QgsField("Northing_RD", QVariant.Double),
])
layer.updateFields()

for element in ifc.by_type("IfcBuildingElement"):
    placement = element.ObjectPlacement
    if not placement:
        continue

    # Get local coordinates from placement (simplified — real extraction
    # requires traversing IfcLocalPlacement chain)
    local_x, local_y, local_z = 0.0, 0.0, 0.0

    # Transform local -> map (EPSG:28992)
    e, n, h = geo.auto_xyz2enh(ifc, local_x, local_y, local_z)

    # Reproject RD New -> WGS84
    lon, lat = transformer.transform(e, n)

    feat = QgsFeature()
    feat.setGeometry(QgsGeometry.fromPointXY(QgsPointXY(lon, lat)))
    feat.setAttributes([
        element.GlobalId,
        element.Name or "",
        element.is_a(),
        e,
        n,
    ])
    provider.addFeature(feat)

layer.updateExtents()
QgsProject.instance().addMapLayer(layer)
```

---

## Example 4: Validate BIM-GIS Coordinate Alignment

Diagnostic script to check if IFC georeferencing produces valid QGIS coordinates.

```python
import ifcopenshell
import ifcopenshell.util.geolocation as geo

def validate_georeferencing(ifc_path):
    """Validate that IFC georeferencing produces plausible map coordinates."""
    ifc = ifcopenshell.open(ifc_path)

    # Check entities exist
    mc_list = ifc.by_type("IfcMapConversion")
    crs_list = ifc.by_type("IfcProjectedCRS")

    if not mc_list:
        return {"valid": False, "error": "Missing IfcMapConversion"}
    if not crs_list:
        return {"valid": False, "error": "Missing IfcProjectedCRS"}

    mc = mc_list[0]
    crs = crs_list[0]

    # Report parameters
    print(f"CRS: {crs.Name}")
    print(f"Eastings: {mc.Eastings}")
    print(f"Northings: {mc.Northings}")
    print(f"OrthogonalHeight: {mc.OrthogonalHeight}")
    print(f"XAxisAbscissa: {mc.XAxisAbscissa}")
    print(f"XAxisOrdinate: {mc.XAxisOrdinate}")
    print(f"Scale: {mc.Scale}")

    # Transform origin (0,0,0) to map coordinates
    e, n, h = geo.auto_xyz2enh(ifc, 0.0, 0.0, 0.0)
    print(f"\nOrigin maps to: E={e:.2f}, N={n:.2f}, H={h:.2f}")

    # Validate against known CRS ranges
    CRS_RANGES = {
        "EPSG:28992": (-7000, 300000, 289000, 629000),
        "EPSG:32631": (166000, 834000, 0, 9400000),
        "EPSG:32632": (166000, 834000, 0, 9400000),
    }

    epsg = crs.Name
    if epsg in CRS_RANGES:
        e_min, e_max, n_min, n_max = CRS_RANGES[epsg]
        e_ok = e_min <= e <= e_max
        n_ok = n_min <= n <= n_max
        if not e_ok or not n_ok:
            return {
                "valid": False,
                "error": f"Coordinates ({e:.0f}, {n:.0f}) outside {epsg} range"
            }

    return {"valid": True, "crs": epsg, "origin_map": (e, n, h)}

# Usage
result = validate_georeferencing("building.ifc")
if result["valid"]:
    print(f"\nGeoreferencing valid. CRS: {result['crs']}")
else:
    print(f"\nGeoreferencing INVALID: {result['error']}")
```

---

## Example 5: Speckle QGIS Connector Approach (Alternative)

Receive BIM data from Speckle server into QGIS. Use ONLY when IfcOpenShell approach is not viable.

```python
# Speckle QGIS Connector — installed via QGIS Plugin Manager
# Plugins > Manage and Install Plugins > search "Speckle"
# Requires: QGIS 3.28.15+, Speckle account

# Manual workflow (no Python API for the QGIS connector):
# 1. Open Speckle panel in QGIS
# 2. Authenticate with Speckle server
# 3. Select project and model
# 4. Click "Receive" to import BIM data as QGIS layers
# 5. Configure CRS via connector settings:
#    - Offset CRS: add X/Y offset to existing CRS
#    - Custom CRS: Transverse Mercator with geographic origin
#    - True North Angle: rotation alignment

# Limitations of Speckle QGIS connector:
# - Legacy connector status (limited active development)
# - Geographic CRS with non-linear units treated as meters
# - Limited receive geometry types (Point, Line, Polyline, Arc, Circle)
# - No IFC entity type preservation — generic geometry only
# - No property set hierarchy — attributes flattened
```

**NOTE**: The Speckle QGIS connector is a legacy connector. ALWAYS prefer the IfcOpenShell + PyQGIS approach for production workflows requiring full geometry and attribute control.
