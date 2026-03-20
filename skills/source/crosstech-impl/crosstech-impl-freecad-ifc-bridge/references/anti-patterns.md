# crosstech-impl-freecad-ifc-bridge — Anti-Patterns Reference

## Anti-Pattern 1: Using Legacy Import Instead of NativeIFC

**Symptom**: IFC data is converted to FreeCAD Part objects, losing property sets, relationships, and parametric geometry.

**Wrong**:
```python
# Legacy import — converts IFC to FreeCAD internal format
import importIFC
importIFC.open("/path/to/building.ifc")
# Result: Part::Feature objects with simplified geometry
# Lost: property sets, spatial relationships, IFC types, material layers
```

**Correct**:
```python
# NativeIFC — IFC file IS the document
import FreeCAD
doc = FreeCAD.openDocument("/path/to/building.ifc")
# Result: IFC entities rendered directly, all data preserved
```

**Why**: Legacy import translates IFC entities into FreeCAD's internal Part representation. This is a lossy conversion. NativeIFC treats the IFC file as the data structure, preserving ALL IFC data.

---

## Anti-Pattern 2: Saving NativeIFC Document as FCStd

**Symptom**: The NativeIFC link is broken. The IFC file is no longer the source of truth.

**Wrong**:
```python
# NativeIFC locked mode document saved as FCStd
doc = FreeCAD.openDocument("/path/to/building.ifc")
doc.saveAs("/path/to/building.FCStd")  # BREAKS NativeIFC link
```

**Correct**:
```python
# Save directly — writes changes to the IFC file
doc = FreeCAD.openDocument("/path/to/building.ifc")
# ... make modifications ...
doc.save()  # Writes to building.ifc
```

**Why**: NativeIFC locked mode means the IFC file IS the document. Saving as `.FCStd` creates a separate file that is no longer synchronized with the IFC. All future edits modify the FCStd, not the IFC.

---

## Anti-Pattern 3: Loading All Shapes for Large Models

**Symptom**: FreeCAD runs out of memory or becomes unresponsive when opening large IFC files (100MB+).

**Wrong**:
```python
doc = FreeCAD.openDocument("/path/to/hospital.ifc")
# Immediately iterate all shapes — loads everything into memory
for obj in doc.Objects:
    volume = obj.Shape.Volume  # Triggers geometry load for EVERY object
```

**Correct**:
```python
doc = FreeCAD.openDocument("/path/to/hospital.ifc")
# Access metadata only first
target_objects = [o for o in doc.Objects
                  if hasattr(o, "IfcType") and o.IfcType == "IfcWall"]
# Load shapes selectively
for obj in target_objects:
    shape = obj.Shape  # Load only walls
```

**Why**: NativeIFC supports lazy geometry loading. Accessing `obj.Shape` triggers geometry computation via IfcOpenShell. Loading all shapes simultaneously for a large model (thousands of objects) exhausts memory. ALWAYS filter objects first, then load shapes for the subset you need.

---

## Anti-Pattern 4: Ignoring IfcOpenShell Version Compatibility

**Symptom**: Silent import failures, missing entities, corrupted geometry.

**Wrong**:
```python
# No version check — assumes any IfcOpenShell works
import FreeCAD
doc = FreeCAD.openDocument("/path/to/model.ifc")
# May fail silently with wrong IfcOpenShell version
```

**Correct**:
```python
import ifcopenshell
import FreeCAD

# ALWAYS verify version compatibility
ios_version = ifcopenshell.version
fc_version = FreeCAD.Version()
print(f"IfcOpenShell: {ios_version}")
print(f"FreeCAD: {fc_version[0]}.{fc_version[1]}")

# FreeCAD 1.0+ requires IfcOpenShell 0.8.x
if not ios_version.startswith("0.8"):
    raise RuntimeError(f"IfcOpenShell {ios_version} may not be compatible with FreeCAD 1.0+")

doc = FreeCAD.openDocument("/path/to/model.ifc")
```

**Why**: FreeCAD bundles a specific IfcOpenShell version. Mismatched versions (e.g., system-installed IfcOpenShell overriding the bundled version) cause silent failures. FreeCAD 1.0+ requires IfcOpenShell 0.8.x. Earlier versions lack NativeIFC support.

---

## Anti-Pattern 5: Expecting FreeCAD Constraints to Survive IFC Export

**Symptom**: Parametric relationships (Sketcher constraints, Part boolean operations) are lost after IFC round-trip.

**Wrong**:
```python
import FreeCAD
import Arch
import Part

# Create wall with Sketcher-based profile
sketch = doc.addObject("Sketcher::SketchObject", "WallProfile")
# ... add constraints ...
wall = Arch.makeWall(sketch, height=3000)

# Export to IFC
import importIFC
importIFC.export([wall], "/path/to/output.ifc")

# Re-import: Sketcher constraints are GONE
importIFC.open("/path/to/output.ifc")
# The wall geometry is correct, but the parametric profile is lost
```

**Correct**:
```python
# Accept that IFC does NOT store FreeCAD-specific constraints
# Option A: Keep the FCStd as the design source, export IFC for exchange
doc.save()  # Save FCStd with constraints
importIFC.export([wall], "/path/to/exchange.ifc")

# Option B: Use NativeIFC from the start (no Sketcher constraints)
# Design directly with IFC-native concepts
doc = FreeCAD.openDocument("/path/to/project.ifc")
```

**Why**: IFC is an exchange format, not a design authoring format. It stores geometry results (BREP, extrusions) but NOT the parametric history (Sketcher constraints, Part booleans, Draft arrays). NEVER expect FreeCAD's internal parametric relationships to survive an IFC round-trip.

---

## Anti-Pattern 6: Mixing Locked and Unlocked NativeIFC Modes

**Symptom**: Confusion about which objects are IFC-managed and which are not. Data inconsistency.

**Wrong**:
```python
# Open IFC in locked mode, then add non-IFC geometry
doc = FreeCAD.openDocument("/path/to/building.ifc")  # Locked mode

# Add a Part box — this does NOT belong in the IFC file
box = doc.addObject("Part::Box", "SiteContext")
box.Length = 10000
# On save, this box may be silently dropped or cause errors
doc.save()
```

**Correct**:
```python
# Use unlocked mode when mixing IFC and non-IFC elements
doc = FreeCAD.newDocument("MixedProject")

# Non-IFC geometry lives in the FreeCAD document
box = doc.addObject("Part::Box", "SiteContext")
box.Length = 10000

# IFC project is attached separately in unlocked mode
# IFC elements link to a separate .ifc file
# Non-IFC elements remain in the .FCStd file
```

**Why**: NativeIFC locked mode means the document IS the IFC file. Adding non-IFC geometry creates objects that have no IFC representation. Use unlocked mode when you need to combine IFC-managed and non-IFC content.

---

## Anti-Pattern 7: Using importIFC.export() in NativeIFC Workflow

**Symptom**: A second, disconnected IFC file is created instead of saving changes to the original.

**Wrong**:
```python
# Open in NativeIFC, then use legacy export
doc = FreeCAD.openDocument("/path/to/building.ifc")
# ... make changes ...

import importIFC
importIFC.export(doc.Objects, "/path/to/building_v2.ifc")
# Creates a NEW IFC file by re-exporting — loses NativeIFC precision
```

**Correct**:
```python
doc = FreeCAD.openDocument("/path/to/building.ifc")
# ... make changes ...
doc.save()  # Writes changes directly to building.ifc
```

**Why**: `importIFC.export()` re-creates IFC entities from FreeCAD objects, which is the legacy pipeline. In NativeIFC mode, the IFC data already exists — `doc.save()` writes modifications directly to the IFC file with minimal changes. Using the legacy exporter on NativeIFC data re-processes everything, potentially losing data.

---

## Anti-Pattern 8: Not Checking Schema Version Before Processing

**Symptom**: IFC4X3 infrastructure entities (IfcAlignment, IfcRoad) are silently dropped or cause errors.

**Wrong**:
```python
import ifcopenshell

ifc_file = ifcopenshell.open("/path/to/infrastructure.ifc")
# Assume IFC4 — query for building elements only
buildings = ifc_file.by_type("IfcBuilding")
# Misses IfcRoad, IfcBridge, IfcAlignment if schema is IFC4X3
```

**Correct**:
```python
import ifcopenshell

ifc_file = ifcopenshell.open("/path/to/infrastructure.ifc")
schema = ifc_file.schema

if schema == "IFC4X3":
    # Query infrastructure entities
    alignments = ifc_file.by_type("IfcAlignment")
    roads = ifc_file.by_type("IfcRoad")
    bridges = ifc_file.by_type("IfcBridge")
    facilities = ifc_file.by_type("IfcFacility")
    print(f"Infrastructure model: {len(alignments)} alignments, "
          f"{len(roads)} roads, {len(bridges)} bridges")
elif schema == "IFC4":
    buildings = ifc_file.by_type("IfcBuilding")
    print(f"Building model: {len(buildings)} buildings")
```

**Why**: IFC4X3 introduced infrastructure entities that do NOT exist in IFC4. Querying `by_type("IfcRoad")` on an IFC4 file raises an error. ALWAYS check the schema version first and handle each schema appropriately.

---

## Anti-Pattern 9: Assuming All Arch Objects Have IfcType Set

**Symptom**: Objects exported to IFC as `IfcBuildingElementProxy` instead of their intended type.

**Wrong**:
```python
import Arch

# Create structure without setting IfcType
column = Arch.makeStructure(length=300, width=300, height=3000)
column.Label = "Column C1"
# column.IfcType is not set — exports as IfcBuildingElementProxy
```

**Correct**:
```python
import Arch

column = Arch.makeStructure(length=300, width=300, height=3000)
column.Label = "Column C1"
column.IfcType = "IfcColumn"  # ALWAYS set explicitly

beam = Arch.makeStructure(length=5000, width=200, height=300)
beam.Label = "Beam B1"
beam.IfcType = "IfcBeam"  # ALWAYS set explicitly
```

**Why**: `Arch.makeStructure()` creates a generic structural element. FreeCAD does NOT automatically detect whether it is a column, beam, or slab. Without an explicit `IfcType`, the element exports as `IfcBuildingElementProxy`, which loses semantic meaning in the IFC model.

---

## Anti-Pattern 10: Editing IFC Files Outside FreeCAD During NativeIFC Session

**Symptom**: Data corruption, lost changes, inconsistent state between FreeCAD viewport and IFC file.

**Wrong**:
```python
# Open in FreeCAD NativeIFC
doc = FreeCAD.openDocument("/path/to/building.ifc")

# Meanwhile, modify the same file with IfcOpenShell in another script
import ifcopenshell
ifc = ifcopenshell.open("/path/to/building.ifc")
wall = ifc.by_type("IfcWall")[0]
wall.Name = "Modified externally"
ifc.write("/path/to/building.ifc")  # Overwrites file while FreeCAD has it open

# Back in FreeCAD — state is now inconsistent
doc.save()  # May overwrite external changes or corrupt the file
```

**Correct**:
```python
# Option A: Close FreeCAD document before external editing
doc.save()
FreeCAD.closeDocument(doc.Name)

# Now safe to edit externally
import ifcopenshell
ifc = ifcopenshell.open("/path/to/building.ifc")
# ... make changes ...
ifc.write("/path/to/building.ifc")

# Re-open in FreeCAD
doc = FreeCAD.openDocument("/path/to/building.ifc")

# Option B: Make all edits within FreeCAD/IfcOpenShell on the same file handle
```

**Why**: NativeIFC locked mode maintains an in-memory representation of the IFC file. External modifications to the file on disk are NOT detected by FreeCAD. Saving from FreeCAD overwrites external changes. ALWAYS close the FreeCAD document before modifying the IFC file externally.
