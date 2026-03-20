---
name: crosstech-impl-freecad-ifc-bridge
description: >
  Use when working with IFC files in FreeCAD, importing BIM models, or doing
  round-trip IFC editing. Prevents the common mistake of using legacy import
  instead of NativeIFC mode or losing IFC data during FreeCAD conversion.
  Covers FreeCAD 1.0+ BIM Workbench, NativeIFC mode, IfcOpenShell integration,
  import/export workflows, and round-trip editing patterns.
  Keywords: FreeCAD, IFC, NativeIFC, BIM Workbench, IfcOpenShell, round-trip,
  FreeCAD BIM, IFC import, IFC export, Arch module.
license: MIT
compatibility: "Designed for Claude Code. Requires FreeCAD 1.0+ with BIM Workbench and IfcOpenShell 0.8.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-impl-freecad-ifc-bridge

## Quick Reference

### FreeCAD IFC Modes

| Mode | FreeCAD Version | IFC Engine | Data Loss | Round-Trip |
|------|----------------|------------|-----------|------------|
| NativeIFC Locked | 1.0+ | IfcOpenShell 0.8.x | NONE | Lossless |
| NativeIFC Unlocked | 1.0+ | IfcOpenShell 0.8.x | NONE (IFC part) | Lossless |
| Legacy Import/Export | < 1.0 / 1.0+ fallback | IfcOpenShell | YES | Lossy |

### BIM Workbench Tools (FreeCAD 1.0+)

| Tool | Purpose |
|------|---------|
| IFC Elements Manager | Control which elements export to IFC |
| IFC Properties Manager | Manage property set attachments |
| IFC Quantities Manager | Handle explicit quantity exports |
| IFC Explorer | Pre-import structural analysis |
| IFC Diff | Visual comparison of two IFC files |

### Critical Warnings

**ALWAYS** use NativeIFC mode for IFC workflows. Legacy import converts IFC to FreeCAD Part objects, losing parametric IFC data permanently.

**NEVER** save an IFC file opened in NativeIFC locked mode as `.FCStd` — the IFC file IS the document. Saving as `.FCStd` breaks the NativeIFC link.

**NEVER** mix NativeIFC locked mode with non-IFC geometry. Use unlocked mode when combining IFC and non-IFC elements in one document.

**ALWAYS** verify IfcOpenShell version compatibility with FreeCAD. Mismatched versions cause silent import failures. Check with `import ifcopenshell; print(ifcopenshell.version)`.

**NEVER** use the legacy `importIFC` module for new projects. Use the BIM Workbench NativeIFC workflow instead.

**ALWAYS** load shapes on demand for large IFC files (100MB+). NativeIFC supports selective geometry loading to manage memory.

---

## Technology Boundary

### Side A: FreeCAD (Part objects, BIM Workbench)

**Version**: FreeCAD 1.0+ (BIM Workbench replaces legacy Arch Workbench)

**Internal representation**:
- `Part::TopoShape` — OpenCASCADE BREP geometry (the native geometry kernel)
- `Arch::Wall`, `Arch::Structure`, `Arch::Window` — parametric BIM objects
- BIM Workbench objects: walls, columns, beams, slabs, doors, windows, pipes, stairs, roofs, panels, equipment
- Properties stored as FreeCAD object properties (`obj.Label`, `obj.IfcType`, custom properties)
- Document format: `.FCStd` (ZIP containing XML + BREP files)

**Scripting API**:
- `FreeCAD` module — document and object management
- `Arch` module — BIM object creation (walls, structures, windows)
- `Part` module — geometry operations (shapes, booleans, extrusions)
- `Draft` module — 2D drawing tools

### Side B: IFC (IfcOpenShell data model)

**Version**: IFC4 (ISO 16739-1:2018), IFC4X3 partial support via IfcOpenShell

**Internal representation**:
- `ifcopenshell.file` — in-memory IFC model
- `entity_instance` — individual IFC entities with typed attributes
- Geometry: CSG, swept solids, BREP, extrusions, tessellation
- Properties: `IfcPropertySet` attached via `IfcRelDefinesByProperties`
- Spatial hierarchy: `IfcProject` > `IfcSite` > `IfcBuilding` > `IfcBuildingStorey`

**Data model**:
- Entities inherit from `IfcRoot` (GlobalId, Name, Description, OwnerHistory)
- Relationships are objectified (`IfcRelAggregates`, `IfcRelContainedInSpatialStructure`)
- Types shared via `IfcRelDefinesByType`
- Materials assigned via `IfcRelAssociatesMaterial`

### The Bridge: NativeIFC / Traditional Import-Export

**NativeIFC (PREFERRED)**: The IFC file IS the FreeCAD document. No conversion happens. IfcOpenShell reads and writes IFC data directly. FreeCAD renders shapes on demand using IfcOpenShell's OpenCASCADE geometry converter.

**Traditional Import**: IfcOpenShell parses IFC geometry and converts it to FreeCAD `Part::TopoShape` objects. This is a one-way conversion that loses IFC semantic data unless explicitly mapped to FreeCAD properties.

**Traditional Export**: FreeCAD BIM objects are serialized to IFC entities using IfcOpenShell. BIM type mappings (`Arch.Wall` -> `IfcWall`) are applied automatically. Non-BIM objects are excluded unless explicitly converted.

**Data flow**:
```
NativeIFC:     IFC file <--direct--> FreeCAD viewport (via IfcOpenShell)
Traditional:   IFC file --> IfcOpenShell --> Part::TopoShape --> FreeCAD
               FreeCAD --> Arch/BIM objects --> IfcOpenShell --> IFC file
```

---

## Critical Rules

1. **ALWAYS** prefer NativeIFC mode over legacy import/export for IFC workflows.
2. **ALWAYS** check IfcOpenShell availability before IFC operations: `import ifcopenshell`.
3. **ALWAYS** use locked NativeIFC mode when the IFC file is the single source of truth.
4. **ALWAYS** use unlocked NativeIFC mode when mixing IFC with non-IFC elements.
5. **ALWAYS** load shapes selectively for large models — NativeIFC supports on-demand loading.
6. **ALWAYS** verify schema version with `ifcopenshell.open(path).schema` before processing.
7. **NEVER** assume FreeCAD preserves IFC parametric geometry in legacy mode — it converts to BREP.
8. **NEVER** edit IFC files outside FreeCAD while they are open in NativeIFC locked mode.
9. **NEVER** rely on FreeCAD-specific parametric constraints surviving IFC export — Sketcher constraints and Part constraints are NOT part of IFC.
10. **NEVER** use `importIFC.export()` for NativeIFC workflows — save the document directly.

---

## Decision Tree

```
START: You need to work with IFC in FreeCAD
|
+-- Q1: Is FreeCAD 1.0+ available?
|   +-- YES --> Use BIM Workbench with NativeIFC
|   +-- NO  --> Use legacy Arch import/export (expect data loss)
|
+-- Q2: What is the workflow?
|   +-- View/inspect IFC --> NativeIFC locked mode (read-only is fine)
|   +-- Edit IFC and save back --> NativeIFC locked mode
|   +-- Combine IFC with non-IFC geometry --> NativeIFC unlocked mode
|   +-- Create BIM model from scratch --> BIM Workbench, export to IFC
|   +-- One-time geometry extraction --> Legacy import is acceptable
|
+-- Q3: Is the IFC file large (>100MB)?
|   +-- YES --> Use NativeIFC with selective shape loading
|   +-- NO  --> Full shape loading is acceptable
|
+-- Q4: Is round-trip fidelity required?
|   +-- YES --> NativeIFC locked mode ONLY
|   +-- NO  --> Legacy export is acceptable for one-way conversion
|
+-- Q5: Do you need scripting access to IFC data?
    +-- YES, to IFC entities --> Use ifcopenshell directly on the file
    +-- YES, to FreeCAD objects --> Use FreeCAD.ActiveDocument.Objects
    +-- YES, to both --> NativeIFC mode (objects ARE IFC entities)
```

---

## Essential Patterns

### Pattern 1: Open IFC in NativeIFC Locked Mode

```python
import FreeCAD

# Open IFC file directly — NativeIFC treats it as the document
doc = FreeCAD.openDocument("/path/to/model.ifc")

# Iterate IFC objects
for obj in doc.Objects:
    if hasattr(obj, "IfcType"):
        print(f"{obj.Label}: {obj.IfcType}")

# Modify an element
wall = doc.getObjectsByLabel("Exterior Wall 01")[0]
wall.Label = "Renamed Wall"

# Save — writes directly to the IFC file, minimal diff
doc.save()
```

### Pattern 2: Access IFC Data via IfcOpenShell (NativeIFC)

```python
import FreeCAD
import ifcopenshell
import ifcopenshell.util.element

# Open the IFC file with IfcOpenShell for semantic queries
ifc_file = ifcopenshell.open("/path/to/model.ifc")

# Query all walls with their properties
for wall in ifc_file.by_type("IfcWall"):
    psets = ifcopenshell.util.element.get_psets(wall)
    name = wall.Name or "Unnamed"
    is_external = psets.get("Pset_WallCommon", {}).get("IsExternal", None)
    print(f"{name}: external={is_external}")

# Check schema version
print(f"Schema: {ifc_file.schema}")  # "IFC2X3", "IFC4", "IFC4X3"
```

### Pattern 3: Create BIM Objects and Export to IFC

```python
import FreeCAD
import Arch
import Draft

# Create a new document
doc = FreeCAD.newDocument("BIM_Project")

# Create BIM objects using the Arch module
wall = Arch.makeWall(length=5000, width=200, height=3000)
wall.IfcType = "IfcWall"
wall.Label = "Exterior Wall 01"

# Create a slab
slab = Arch.makeStructure(length=6000, width=5000, height=200)
slab.IfcType = "IfcSlab"
slab.Label = "Ground Floor Slab"

# Recompute to update geometry
doc.recompute()

# Export to IFC using the legacy exporter
import importIFC
importIFC.export([wall, slab], "/path/to/output.ifc")
```

### Pattern 4: NativeIFC Unlocked Mode (Mixed Content)

```python
import FreeCAD
import Arch

# Create document with mixed content
doc = FreeCAD.newDocument("Mixed_Project")

# Add non-IFC geometry (parametric FreeCAD part)
import Part
box = doc.addObject("Part::Box", "SiteContext")
box.Length = 10000
box.Width = 10000
box.Height = 100

# Add IFC project (unlocked mode — IFC and non-IFC coexist)
# The IFC project attaches to a separate .ifc file
# Non-IFC objects (like the box) remain in the .FCStd file

doc.recompute()
```

### Pattern 5: Selective Shape Loading for Large Models

```python
import FreeCAD

# Open large IFC file — NativeIFC loads metadata only
doc = FreeCAD.openDocument("/path/to/large_model.ifc")

# Shapes are loaded on demand per object
# Access an object to trigger shape loading
obj = doc.getObjectsByLabel("Foundation Wall")[0]
shape = obj.Shape  # Triggers geometry computation via IfcOpenShell

# For batch processing without visualization, avoid loading shapes
for obj in doc.Objects:
    if hasattr(obj, "IfcType"):
        # Access metadata only — no shape loading
        print(f"{obj.Label}: {obj.IfcType}")
```

---

## Common Operations

### Import/Export Operations

| Operation | NativeIFC (Preferred) | Legacy |
|-----------|----------------------|--------|
| Open IFC | `FreeCAD.openDocument("f.ifc")` | File > Import > IFC |
| Save IFC | `doc.save()` (writes to .ifc) | `importIFC.export(objs, "f.ifc")` |
| Access properties | Via IfcOpenShell on same file | Custom properties on FreeCAD object |
| Round-trip edit | Direct — changes write to IFC | Lossy — re-export required |
| Large file handling | Selective shape loading | Full load into memory |

### FreeCAD BIM Type Mapping

| FreeCAD BIM Object | IFC Entity | Notes |
|-------------------|------------|-------|
| `Arch.makeWall()` | `IfcWall` | Maps directly |
| `Arch.makeStructure()` | `IfcColumn` / `IfcBeam` / `IfcSlab` | Based on IfcType property |
| `Arch.makeWindow()` | `IfcWindow` / `IfcDoor` | Based on IfcType property |
| `Arch.makeStairs()` | `IfcStair` | Maps directly |
| `Arch.makeRoof()` | `IfcRoof` | Maps directly |
| `Arch.makePipe()` | `IfcPipeSegment` | Maps directly |
| `Arch.makeSite()` | `IfcSite` | Spatial element |
| `Arch.makeBuilding()` | `IfcBuilding` | Spatial element |
| `Arch.makeFloor()` | `IfcBuildingStorey` | Spatial element |

### Data Preservation Matrix

| Data Type | NativeIFC | Legacy Import | Legacy Export |
|-----------|:---------:|:------------:|:------------:|
| IFC entity class | FULL | FULL | FULL |
| GlobalId | FULL | Stored as property | Generated new |
| Property sets | FULL | Partial (custom props) | Partial |
| Quantities | FULL | Partial | Partial |
| Relationships | FULL | Simplified | Simplified |
| Parametric geometry | FULL | Converted to BREP | From BREP |
| Material layers | FULL | Simplified | Simplified |
| Spatial hierarchy | FULL | Collection hierarchy | Regenerated |
| FreeCAD constraints | N/A | N/A | **LOST** |
| Sketcher data | N/A | N/A | **LOST** |

### Scripting Quick Reference

```python
# Check FreeCAD version
import FreeCAD
print(FreeCAD.Version())  # ['1', '0', '0', ...]

# Check IfcOpenShell version
import ifcopenshell
print(ifcopenshell.version)  # "0.8.0" or similar

# Check if NativeIFC is available
try:
    import nativeifc
    NATIVEIFC_AVAILABLE = True
except ImportError:
    NATIVEIFC_AVAILABLE = False

# Get all IFC objects in current document
doc = FreeCAD.ActiveDocument
ifc_objects = [o for o in doc.Objects if hasattr(o, "IfcType")]

# Access underlying IfcOpenShell entity (NativeIFC)
import ifcopenshell
ifc_file = ifcopenshell.open(doc.FileName)
entity = ifc_file.by_id(42)  # Access by expressID
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- FreeCAD BIM/Arch API and IfcOpenShell within FreeCAD
- [references/examples.md](references/examples.md) -- Import/export/round-trip workflow examples
- [references/anti-patterns.md](references/anti-patterns.md) -- FreeCAD IFC mistakes and data loss patterns

### Official Sources

- https://wiki.freecad.org/BIM_Workbench -- FreeCAD BIM Workbench documentation
- https://wiki.freecad.org/NativeIFC -- NativeIFC mode documentation
- https://github.com/yorikvanhavre/FreeCAD-NativeIFC -- NativeIFC source code
- https://docs.ifcopenshell.org/ -- IfcOpenShell 0.8.x API documentation
- https://wiki.freecad.org/Arch_IFC -- Legacy Arch IFC import/export
- https://wiki.freecad.org/Manual:BIM_modeling -- FreeCAD BIM modeling manual
