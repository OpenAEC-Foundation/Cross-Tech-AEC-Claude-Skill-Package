# crosstech-impl-qgis-bim-georef — Anti-Patterns

## Anti-Pattern 1: Importing IFC Without Checking Georeferencing

**What happens**: IFC geometry is placed at local coordinates (near 0,0,0). Without IfcMapConversion, these coordinates are interpreted as map coordinates in whatever CRS QGIS uses, placing the building near (0,0) — typically in the Gulf of Guinea for EPSG:4326.

**Wrong:**
```python
# Extracting geometry and dumping directly to GeoDataFrame
# without checking for IfcMapConversion
gdf = gpd.GeoDataFrame(features, crs="EPSG:28992")  # Coordinates are WRONG
```

**Correct:**
```python
mc_list = ifc.by_type("IfcMapConversion")
crs_list = ifc.by_type("IfcProjectedCRS")

if not mc_list or not crs_list:
    raise ValueError("IFC file lacks georeferencing — cannot place on map")

# Transform coordinates BEFORE creating GeoDataFrame
e, n, h = geo.auto_xyz2enh(ifc, local_x, local_y, local_z)
```

**Why**: IfcMapConversion provides the translation, rotation, and scale needed to convert local BIM coordinates to real-world map coordinates. Without it, coordinates are meaningless in a GIS context.

---

## Anti-Pattern 2: Using GDAL to Read IFC Files

**What happens**: GDAL does NOT have a native IFC driver (verified: GDAL 3.9). Attempting to use `ogr2ogr` or QGIS "Add Vector Layer" with an IFC file fails silently or produces no output.

**Wrong:**
```python
# This does NOT work — GDAL has no IFC driver
layer = QgsVectorLayer("building.ifc", "BIM Data", "ogr")
# layer.isValid() returns False
```

```bash
# This does NOT work
ogr2ogr -f GPKG output.gpkg building.ifc
```

**Correct:**
```python
# Use IfcOpenShell to extract geometry, then load via GeoPackage
import ifcopenshell
# ... extract and transform geometry ...
gdf.to_file("building.gpkg", driver="GPKG")
layer = QgsVectorLayer("building.gpkg", "BIM Data", "ogr")
```

**Why**: The GDAL vector driver list (https://gdal.org/en/stable/drivers/vector/index.html) does NOT include IFC. The community project `ogr2ifc` works in the reverse direction (GIS -> IFC) only.

---

## Anti-Pattern 3: Using Shapefile Format for BIM-GIS Export

**What happens**: Shapefiles drop Z-coordinates, truncate field names to 10 characters, and limit file size to 2 GB. BIM attribute names like `Pset_WallCommon.FireRating` become meaningless truncated strings.

**Wrong:**
```python
gdf.to_file("building.shp", driver="ESRI Shapefile")  # Z values LOST
```

**Correct:**
```python
gdf.to_file("building.gpkg", driver="GPKG")  # Z preserved, long field names OK
```

**Why**: GeoPackage supports Z and M dimensions, field names up to 256 characters, spatial indexing, and multiple layers in one file. It is the recommended OGC standard for vector GIS data.

---

## Anti-Pattern 4: Projecting All Storeys to 2D Without Filtering

**What happens**: Multi-storey building elements project onto the same 2D area, creating overlapping polygons. A 10-storey building produces 10 overlapping footprints for each column position. Analysis results (area calculations, spatial queries) become meaningless.

**Wrong:**
```python
# Processing ALL building elements without storey filtering
for element in ifc.by_type("IfcBuildingElement"):
    shape = ifcopenshell.geom.create_shape(settings, element)
    # Creates overlapping footprints from different storeys
```

**Correct:**
```python
# Filter by target storey first
target_storey = ifc.by_type("IfcBuildingStorey")[0]  # Ground floor
for rel in target_storey.ContainsElements:
    for element in rel.RelatedElements:
        shape = ifcopenshell.geom.create_shape(settings, element)
        # Only ground floor elements — no overlaps
```

**Why**: BIM models are inherently 3D. QGIS is primarily a 2D system. The 3D-to-2D projection collapses the vertical dimension, merging all storeys into one plane.

---

## Anti-Pattern 5: Ignoring Unit Mismatch Between IFC and CRS

**What happens**: IFC files using millimeters produce coordinates 1000x larger than expected when `IfcMapConversion.Scale` is not set correctly. A building at 155000 m easting becomes 155000000 in the GIS, placing it far outside any valid CRS range.

**Wrong:**
```python
# Assuming project units are meters without checking
e = local_x + eastings  # If local_x is in mm, this is WRONG
```

**Correct:**
```python
# Check IFC project units
units = ifc.by_type("IfcUnitAssignment")[0]
for unit in units.Units:
    if hasattr(unit, "UnitType") and unit.UnitType == "LENGTHUNIT":
        if hasattr(unit, "Prefix") and unit.Prefix == "MILLI":
            print("Project uses millimeters — Scale should be 0.001")

# Use auto_xyz2enh which reads Scale from IfcMapConversion
e, n, h = geo.auto_xyz2enh(ifc, local_x, local_y, local_z)
```

**Why**: IfcMapConversion.Scale converts from IFC project units to CRS map units. When the IFC uses mm and the CRS uses m, Scale MUST be 0.001. Omitting Scale defaults to 1.0, making everything 1000x too large.

---

## Anti-Pattern 6: Using create_shape() in a Loop Instead of Iterator

**What happens**: Calling `ifcopenshell.geom.create_shape()` individually for each element is significantly slower than using the geometry iterator. For large IFC files (100MB+), this turns a minutes-long operation into an hours-long operation.

**Wrong:**
```python
for element in ifc.by_type("IfcBuildingElement"):
    shape = ifcopenshell.geom.create_shape(settings, element)  # SLOW
```

**Correct:**
```python
import multiprocessing

iterator = ifcopenshell.geom.iterator(
    settings, ifc, multiprocessing.cpu_count()
)
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = ifc.by_id(shape.id)
        # ... process ...
        if not iterator.next():
            break
```

**Why**: The geometry iterator uses C++ multithreading internally and caches shared geometry definitions. Individual `create_shape()` calls re-initialize the geometry kernel for each element.

---

## Anti-Pattern 7: Omitting always_xy=True in pyproj

**What happens**: EPSG:4326 (WGS84) officially uses (latitude, longitude) axis order, not (longitude, latitude). Without `always_xy=True`, pyproj follows the official order, causing silent coordinate swaps: a point in Amsterdam (lon=4.9, lat=52.4) becomes (52.4, 4.9), placing it in Kazakhstan.

**Wrong:**
```python
transformer = Transformer.from_crs("EPSG:28992", "EPSG:4326")
result = transformer.transform(155000, 463000)
# result = (52.15, 5.38) — that's (lat, lon), NOT (lon, lat)
```

**Correct:**
```python
transformer = Transformer.from_crs("EPSG:28992", "EPSG:4326", always_xy=True)
lon, lat = transformer.transform(155000, 463000)
# lon=5.38, lat=52.15 — correct order for mapping
```

**Why**: The EPSG registry defines EPSG:4326 with (latitude, longitude) axis order. The `always_xy=True` parameter forces (x=longitude, y=latitude) order, matching the convention used by QGIS, web maps, and GeoJSON.

---

## Anti-Pattern 8: Mixing NAP and Ellipsoidal Heights

**What happens**: In the Netherlands, the geoid undulation between NAP (Normaal Amsterdams Peil) and WGS84 ellipsoidal height is approximately 42-44 meters. Mixing them causes buildings to appear floating above or sunk into terrain.

**Wrong:**
```python
# Using NAP height directly as WGS84 ellipsoidal height
elevation_wgs84 = nap_height  # Off by ~42m in the Netherlands
```

**Correct:**
```python
from pyproj import Transformer

# Use compound CRS EPSG:7415 (RD New + NAP) for proper 3D transformation
transformer = Transformer.from_crs(
    "EPSG:7415",   # RD New + NAP height
    "EPSG:4979",   # WGS84 3D (geographic + ellipsoidal height)
    always_xy=True
)
lon, lat, ellipsoidal_h = transformer.transform(easting, northing, nap_height)
```

**Why**: NAP is a physical height system (based on mean sea level). WGS84 ellipsoidal height is a mathematical height above the reference ellipsoid. The difference (geoid undulation) varies by location and is ~42-44m in the Netherlands.

---

## Anti-Pattern 9: Assigning Wrong CRS to QGIS Layer

**What happens**: If the IFC file uses EPSG:28992 (RD New) but you assign EPSG:32631 (UTM 31N) to the QGIS layer, the building appears hundreds of kilometers from its actual location. Both CRS use meters, so the coordinates look plausible but are wrong.

**Wrong:**
```python
# Guessing the CRS instead of reading from IfcProjectedCRS
layer = QgsVectorLayer("Point?crs=EPSG:32631", "BIM Data", "memory")
# Coordinates from EPSG:28992 are now misinterpreted as UTM 31N
```

**Correct:**
```python
# Read CRS from IFC file
crs_entity = ifc.by_type("IfcProjectedCRS")[0]
epsg = crs_entity.Name  # "EPSG:28992"
layer = QgsVectorLayer(f"Point?crs={epsg}", "BIM Data", "memory")
```

**Detection**: If the model appears in the right country but at a wrong position, check whether RD New coordinates were interpreted as UTM or vice versa. RD New eastings (0-300k) and UTM eastings (166k-834k) have overlapping ranges, making this error hard to spot without visual verification on a basemap.

---

## Diagnostic Checklist

When BIM data appears at the wrong location in QGIS, check in this order:

| Step | Check | Symptom if Wrong |
|------|-------|-----------------|
| 1 | IfcMapConversion exists? | Model at (0,0) / Gulf of Guinea |
| 2 | IfcProjectedCRS.Name correct? | Hundreds of km off position |
| 3 | IFC project units match Scale? | 1000x too large or small |
| 4 | XAxisAbscissa/Ordinate correct? | Building rotated on map |
| 5 | always_xy=True in pyproj? | Coordinates swapped (lat/lon) |
| 6 | NAP vs ellipsoidal height? | ~42m vertical offset |
| 7 | Correct UTM zone? | ~500km east-west offset |
