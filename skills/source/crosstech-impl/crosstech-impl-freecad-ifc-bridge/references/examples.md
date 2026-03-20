# crosstech-impl-freecad-ifc-bridge — Examples Reference

## Example 1: Open and Inspect IFC in NativeIFC Mode

Open an IFC file in FreeCAD NativeIFC locked mode and inspect its contents.

```python
import FreeCAD
import ifcopenshell
import ifcopenshell.util.element

# Open IFC file — NativeIFC treats it as the document
doc = FreeCAD.openDocument("/path/to/building.ifc")

# List all objects with their IFC types
for obj in doc.Objects:
    if hasattr(obj, "IfcType"):
        print(f"  {obj.Label} [{obj.IfcType}]")

# Access IFC data via IfcOpenShell for semantic queries
ifc_file = ifcopenshell.open(doc.FileName)
print(f"Schema: {ifc_file.schema}")
print(f"Total products: {len(ifc_file.by_type('IfcProduct'))}")

# Get all walls with their property sets
for wall in ifc_file.by_type("IfcWall"):
    psets = ifcopenshell.util.element.get_psets(wall)
    container = ifcopenshell.util.element.get_container(wall)
    storey_name = container.Name if container else "No container"
    print(f"  {wall.Name} in {storey_name}")
    for pset_name, props in psets.items():
        print(f"    {pset_name}: {len(props)} properties")
```

## Example 2: Round-Trip Edit in NativeIFC Locked Mode

Edit an IFC file in place using NativeIFC — changes write directly to the IFC file with minimal diff.

```python
import FreeCAD

# Open IFC in NativeIFC locked mode
doc = FreeCAD.openDocument("/path/to/building.ifc")

# Find a wall by label
walls = doc.getObjectsByLabel("Exterior Wall 01")
if not walls:
    raise ValueError("Wall not found")
wall = walls[0]

# Modify properties
wall.Label = "Exterior Wall 01 - Modified"

# Modify geometry (if supported by the NativeIFC implementation)
# Note: graphical editing is limited in FreeCAD 1.0
# Use IfcOpenShell for complex modifications

# Recompute and save — writes directly to the IFC file
doc.recompute()
doc.save()

# Verify: the IFC file now contains the modification
# Use IFC Diff tool to see exactly what changed
```

## Example 3: Create BIM Model from Scratch and Export to IFC

Build a simple building in FreeCAD and export it to IFC.

```python
import FreeCAD
import Arch
import Draft

# Create new document
doc = FreeCAD.newDocument("SimpleBuilding")

# Create spatial hierarchy
site = Arch.makeSite(name="Construction Site")
building = Arch.makeBuilding(name="Office Building")
ground_floor = Arch.makeFloor(name="Ground Floor")
first_floor = Arch.makeFloor(name="First Floor")

# Set floor heights
ground_floor.Placement.Base.z = 0
first_floor.Placement.Base.z = 3000

# Create walls for ground floor
wall1 = Arch.makeWall(length=6000, width=200, height=3000)
wall1.Label = "North Wall"
wall1.IfcType = "IfcWall"

wall2 = Arch.makeWall(length=4000, width=200, height=3000)
wall2.Label = "East Wall"
wall2.IfcType = "IfcWall"
wall2.Placement.Base.x = 6000
wall2.Placement.Rotation = FreeCAD.Rotation(FreeCAD.Vector(0, 0, 1), 90)

# Create a slab
slab = Arch.makeStructure(length=6000, width=4000, height=200)
slab.Label = "Ground Floor Slab"
slab.IfcType = "IfcSlab"

# Create a column
column = Arch.makeStructure(length=300, width=300, height=3000)
column.Label = "Column C1"
column.IfcType = "IfcColumn"
column.Placement.Base = FreeCAD.Vector(3000, 2000, 0)

# Organize hierarchy
ground_floor.Group = [wall1, wall2, slab, column]
building.Group = [ground_floor, first_floor]
site.Group = [building]

# Recompute
doc.recompute()

# Export to IFC
import importIFC
importIFC.export(doc.Objects, "/path/to/output.ifc")

print("Export complete")
```

## Example 4: Query IFC Properties with IfcOpenShell

Extract structured property data from an IFC file opened in FreeCAD.

```python
import ifcopenshell
import ifcopenshell.util.element
import ifcopenshell.util.unit

# Open IFC file
ifc_file = ifcopenshell.open("/path/to/building.ifc")

# ALWAYS check units first
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(ifc_file)
print(f"Unit scale to meters: {unit_scale}")

# Extract wall schedule data
print("WALL SCHEDULE")
print("-" * 60)
for wall in ifc_file.by_type("IfcWall"):
    # Get properties from BOTH occurrence AND type
    psets = ifcopenshell.util.element.get_psets(wall)

    # Get type information
    wall_type = ifcopenshell.util.element.get_type(wall)
    type_name = wall_type.Name if wall_type else "No type"

    # Get spatial container
    container = ifcopenshell.util.element.get_container(wall)
    storey = container.Name if container else "Unplaced"

    # Get common properties
    common = psets.get("Pset_WallCommon", {})
    is_external = common.get("IsExternal", "Unknown")
    is_loadbearing = common.get("LoadBearing", "Unknown")
    fire_rating = common.get("FireRating", "N/A")

    print(f"  {wall.Name}")
    print(f"    Type: {type_name}")
    print(f"    Storey: {storey}")
    print(f"    External: {is_external}")
    print(f"    Load-bearing: {is_loadbearing}")
    print(f"    Fire rating: {fire_rating}")
    print()
```

## Example 5: Selective Shape Loading for Large Models

Handle large IFC files (100MB+) by loading geometry on demand.

```python
import FreeCAD

# Open large IFC file — NativeIFC loads metadata only
doc = FreeCAD.openDocument("/path/to/large_hospital.ifc")

# Count objects without loading geometry
total = len(doc.Objects)
ifc_objects = [o for o in doc.Objects if hasattr(o, "IfcType")]
print(f"Total objects: {total}, IFC objects: {len(ifc_objects)}")

# Count by type without loading shapes
type_counts = {}
for obj in ifc_objects:
    ifc_type = obj.IfcType
    type_counts[ifc_type] = type_counts.get(ifc_type, 0) + 1

for ifc_type, count in sorted(type_counts.items()):
    print(f"  {ifc_type}: {count}")

# Load shapes ONLY for specific objects you need
target_walls = [o for o in ifc_objects if o.IfcType == "IfcWall"]
for wall in target_walls[:10]:  # Load first 10 walls only
    shape = wall.Shape  # Triggers geometry computation
    if shape.isValid():
        volume = shape.Volume
        print(f"  {wall.Label}: volume = {volume:.2f} mm3")
```

## Example 6: Batch Property Modification via IfcOpenShell

Modify IFC properties programmatically using IfcOpenShell directly.

```python
import ifcopenshell
import ifcopenshell.api

# Open IFC file
ifc_file = ifcopenshell.open("/path/to/building.ifc")

# Add a custom property set to all walls
for wall in ifc_file.by_type("IfcWall"):
    # Create property set using IfcOpenShell API
    pset = ifcopenshell.api.run("pset.add_pset", ifc_file,
                                 product=wall,
                                 name="Custom_WallData")

    # Add properties to the set
    ifcopenshell.api.run("pset.edit_pset", ifc_file,
                          pset=pset,
                          properties={
                              "InspectionDate": "2025-01-15",
                              "Inspector": "Automated Script",
                              "ConditionScore": 8
                          })

# Save modified file
ifc_file.write("/path/to/building_updated.ifc")
print("Properties added to all walls")
```

## Example 7: Convert Legacy FreeCAD BIM to NativeIFC

Migrate a legacy FreeCAD BIM document to NativeIFC workflow.

```python
import FreeCAD
import Arch
import importIFC

# Step 1: Open legacy FreeCAD document
doc = FreeCAD.openDocument("/path/to/legacy_project.FCStd")

# Step 2: Collect all BIM objects
bim_objects = []
for obj in doc.Objects:
    if hasattr(obj, "IfcType") and obj.IfcType:
        bim_objects.append(obj)
    elif obj.isDerivedFrom("Arch::Wall"):
        obj.IfcType = "IfcWall"
        bim_objects.append(obj)
    elif obj.isDerivedFrom("Arch::Structure"):
        if not obj.IfcType:
            obj.IfcType = "IfcBuildingElementProxy"
        bim_objects.append(obj)

print(f"Found {len(bim_objects)} BIM objects")

# Step 3: Export to IFC
importIFC.export(bim_objects, "/path/to/migrated_project.ifc")

# Step 4: Re-open in NativeIFC mode
FreeCAD.closeDocument(doc.Name)
native_doc = FreeCAD.openDocument("/path/to/migrated_project.ifc")

# Step 5: Verify migration
for obj in native_doc.Objects:
    if hasattr(obj, "IfcType"):
        print(f"  {obj.Label}: {obj.IfcType}")

print("Migration to NativeIFC complete")
# From now on, work directly with the .ifc file
```

## Example 8: FreeCAD IFC to IfcOpenShell Round-Trip Validation

Validate that round-trip editing preserves IFC data integrity.

```python
import ifcopenshell
import ifcopenshell.util.element

def compare_ifc_files(path_before, path_after):
    """Compare two IFC files to detect data loss."""
    before = ifcopenshell.open(path_before)
    after = ifcopenshell.open(path_after)

    # Compare entity counts
    for ifc_type in ["IfcWall", "IfcColumn", "IfcSlab", "IfcBeam",
                     "IfcDoor", "IfcWindow", "IfcSpace"]:
        count_before = len(before.by_type(ifc_type))
        count_after = len(after.by_type(ifc_type))
        status = "OK" if count_before == count_after else "MISMATCH"
        print(f"  {ifc_type}: {count_before} -> {count_after} [{status}]")

    # Compare property sets on walls
    walls_before = {w.GlobalId: w for w in before.by_type("IfcWall")}
    walls_after = {w.GlobalId: w for w in after.by_type("IfcWall")}

    common_ids = set(walls_before.keys()) & set(walls_after.keys())
    for guid in common_ids:
        psets_before = ifcopenshell.util.element.get_psets(walls_before[guid])
        psets_after = ifcopenshell.util.element.get_psets(walls_after[guid])
        if set(psets_before.keys()) != set(psets_after.keys()):
            print(f"  Wall {guid}: property set MISMATCH")
            print(f"    Before: {sorted(psets_before.keys())}")
            print(f"    After:  {sorted(psets_after.keys())}")

    # Check for lost entities
    lost_ids = set(walls_before.keys()) - set(walls_after.keys())
    if lost_ids:
        print(f"  WARNING: {len(lost_ids)} walls lost during round-trip")

    print("Validation complete")

# Usage
compare_ifc_files("/path/to/original.ifc", "/path/to/after_edit.ifc")
```
