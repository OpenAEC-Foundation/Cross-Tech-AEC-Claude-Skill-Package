# Vooronderzoek: IFC Ecosystem for Cross-Technology Integration

> Research document for the Cross-Tech AEC Skill Package.
> Focus: IFC schema structure, tooling APIs, and cross-technology mapping.
> Date: 2026-03-20
> Status: Complete — verified against official documentation where noted.

---

## 1. IFC Schema Structure

### 1.1 Schema Versions

The Industry Foundation Classes (IFC) standard is maintained by buildingSMART International. The three schema versions relevant to modern AEC workflows are:

| Version | ISO Standard | Status | Primary Domain |
|---------|-------------|--------|----------------|
| IFC2X3 | ISO/PAS 16739:2005 | Legacy, widely deployed | Buildings |
| IFC4 ADD2 TC1 | ISO 16739-1:2018 | Current production standard | Buildings |
| IFC4X3 (4.3.2) | ISO 16739-1:2024 | Current, infrastructure-extended | Buildings + Infrastructure |

**Verified against**: [IFC4 ADD2 TC1 schema](https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/) and [IFC4X3 documentation](https://ifc43-docs.standards.buildingsmart.org/).

### 1.2 Entity Hierarchy

The IFC schema is rooted in a single abstract supertype. The inheritance chain is:

```
IfcRoot (abstract)
├── IfcObjectDefinition (abstract)
│   ├── IfcObject (abstract) — individual occurrences
│   │   ├── IfcProduct — physical/spatial objects with placement + shape
│   │   │   ├── IfcElement — building elements (walls, slabs, beams)
│   │   │   │   ├── IfcBuildingElement (IfcWall, IfcSlab, IfcBeam, IfcColumn, IfcDoor, IfcWindow, ...)
│   │   │   │   ├── IfcDistributionElement (HVAC, plumbing, electrical)
│   │   │   │   ├── IfcOpeningElement — voids cut into elements
│   │   │   │   └── IfcFurnishingElement — furniture
│   │   │   ├── IfcSpatialElement — spatial hierarchy containers
│   │   │   │   ├── IfcSite
│   │   │   │   ├── IfcBuilding
│   │   │   │   ├── IfcBuildingStorey
│   │   │   │   └── IfcSpace
│   │   │   └── IfcAnnotation, IfcGrid, IfcPort, ...
│   │   ├── IfcProcess — time-based actions (tasks, events, procedures)
│   │   ├── IfcControl — constraints (cost items, schedules, permits)
│   │   ├── IfcResource — labor, material, equipment resources
│   │   ├── IfcActor — human agents
│   │   └── IfcGroup — arbitrary object collections
│   ├── IfcTypeObject — shared type definitions
│   │   └── IfcTypeProduct
│   │       └── IfcElementType (IfcWallType, IfcSlabType, ...)
│   └── IfcContext
│       └── IfcProject — top-level project container
├── IfcRelationship (abstract) — objectified relationships
│   ├── IfcRelDecomposes
│   │   └── IfcRelAggregates — whole/part (Project→Site→Building→Storey)
│   ├── IfcRelAssigns
│   │   └── IfcRelAssignsToGroup, IfcRelAssignsToProduct, ...
│   ├── IfcRelConnects
│   │   └── IfcRelContainedInSpatialStructure — element→storey containment
│   ├── IfcRelDefines
│   │   ├── IfcRelDefinesByProperties — attaches property sets to objects
│   │   └── IfcRelDefinesByType — links occurrence to type
│   ├── IfcRelAssociates
│   │   └── IfcRelAssociatesMaterial, IfcRelAssociatesClassification, ...
│   └── IfcRelDeclares — links definitions to project context
└── IfcPropertyDefinition (abstract)
    ├── IfcPropertySet — named collection of properties
    └── IfcPropertySetTemplate — schema for property sets
```

**Verified against**: IFC4 ADD2 TC1 kernel schema documentation.

### 1.3 Key Relationship Types

IFC uses "objectified relationships" — relationships are entities themselves, not just foreign keys. This is a critical concept for cross-technology integration because many target formats (JSON, Three.js scene graphs, Blender object trees) do NOT have objectified relationships. This is the **number one source of data loss** when converting IFC to other formats.

| Relationship | Purpose | Source → Target |
|-------------|---------|-----------------|
| `IfcRelAggregates` | Spatial decomposition | IfcProject → IfcSite → IfcBuilding → IfcBuildingStorey |
| `IfcRelContainedInSpatialStructure` | Element containment | IfcBuildingStorey → [IfcWall, IfcSlab, ...] |
| `IfcRelDefinesByProperties` | Property attachment | IfcProduct → IfcPropertySet |
| `IfcRelDefinesByType` | Type assignment | IfcWall → IfcWallType |
| `IfcRelAssociatesMaterial` | Material assignment | IfcProduct → IfcMaterialLayerSetUsage |
| `IfcRelVoidsElement` | Opening/void cutting | IfcOpeningElement → IfcWall |
| `IfcRelFillsElement` | Fill opening | IfcDoor → IfcOpeningElement |
| `IfcRelSpaceBoundary` | Space boundary | IfcSpace → IfcWall (thermal analysis) |

### 1.4 Property Sets and Quantity Sets

**Property Sets (IfcPropertySet)**:
- Named containers grouping `IfcPropertySingleValue`, `IfcPropertyEnumeratedValue`, `IfcPropertyBoundedValue`, `IfcPropertyListValue`, `IfcPropertyTableValue` instances.
- Attached to objects via `IfcRelDefinesByProperties`.
- Standard property sets are prefixed `Pset_` (e.g., `Pset_WallCommon`, `Pset_DoorCommon`).
- Custom property sets use any name.
- Each `IfcPropertySingleValue` has a `Name`, `NominalValue` (typed via `IfcValue` subtypes: `IfcLabel`, `IfcReal`, `IfcInteger`, `IfcBoolean`, `IfcText`), and optional `Unit`.

**Quantity Sets (IfcElementQuantity)**:
- Named containers grouping `IfcQuantityLength`, `IfcQuantityArea`, `IfcQuantityVolume`, `IfcQuantityCount`, `IfcQuantityWeight`, `IfcQuantityTime`.
- Standard quantity sets are prefixed `Qto_` (e.g., `Qto_WallBaseQuantities`).
- Attached to objects via the same `IfcRelDefinesByProperties` mechanism.
- Each quantity has a `Name`, a numeric value, and an optional `Unit` and `Formula`.

**Type Objects**:
- `IfcTypeObject` / `IfcElementType` defines shared characteristics.
- Property sets can be attached to the type (shared by all instances) or to individual occurrences.
- Linked via `IfcRelDefinesByType`.
- When querying properties, you MUST check both the occurrence AND its type to get the complete property picture.

### 1.5 IFC4 vs IFC4X3 — Key Differences

IFC4X3 (ISO 16739-1:2024) extends IFC4 with infrastructure support. Key additions:

**New Entity Types**:
- `IfcAlignment` — linear referencing for roads/railways (subtypes: `IfcAlignmentHorizontal`, `IfcAlignmentVertical`, `IfcAlignmentCant`)
- `IfcRoad`, `IfcRailway`, `IfcBridge`, `IfcMarineFacility` — infrastructure spatial elements
- `IfcCourse`, `IfcPavement`, `IfcKerb` — road-specific elements
- `IfcEarthworksCut`, `IfcEarthworksFill` — earthworks elements
- `IfcBorehole`, `IfcGeomodel`, `IfcGeoslice`, `IfcSolidStratum` — geotechnical elements
- `IfcLinearElement`, `IfcLinearPositioningElement` — linear infrastructure positioning
- `IfcFacilityPart` — generalized spatial decomposition for infrastructure

**Schema Changes**:
- Enhanced spatial structure: `IfcFacility` as generalized supertype (IfcBuilding, IfcBridge, IfcRoad, IfcRailway are subtypes).
- Advanced geometry: sweep operations with cant, advanced surfaces for road design.
- `IfcSpatialStructureElement` deprecated in favor of `IfcSpatialElement` hierarchy.
- New relationship: `IfcRelPositions` for linear positioning.

**Breaking Changes for Integration**:
- Tools supporting only IFC4 WILL NOT recognize IFC4X3 infrastructure entities — they appear as unknown types.
- IfcOpenShell v0.7.x supports IFC4X3 partially; v0.8.x has improved support.
- web-ifc support for IFC4X3 is evolving; geometry for alignment entities needs verification.
- Schema version mismatch is a common failure mode: an IFC4X3 file opened in an IFC4-only tool will lose infrastructure entities silently.

**Verified against**: [IFC4X3 documentation](https://ifc43-docs.standards.buildingsmart.org/) and [buildingSMART forums](https://forums.buildingsmart.org/t/ifc-4x3-vs-4-1-changes/3217).

---

## 2. IfcOpenShell Python API

### 2.1 Overview

IfcOpenShell is the primary open-source IFC toolkit. It provides a C++ geometry engine with Python bindings. Current version: **0.8.4** (as of documentation).

**Verified against**: [IfcOpenShell 0.8.4 documentation](https://docs.ifcopenshell.org/).

### 2.2 Core Classes

**`ifcopenshell.file`** — The main file object representing an IFC model in memory.

```python
import ifcopenshell

# Open existing file
model = ifcopenshell.open("model.ifc")

# Create blank file (default schema: IFC4)
model = ifcopenshell.file(schema="IFC4")
# Supported schemas: "IFC2X3", "IFC4", "IFC4X3"
```

Key methods on `ifcopenshell.file`:

| Method | Signature | Purpose |
|--------|-----------|---------|
| `by_type()` | `by_type(type: str, include_subtypes=True) -> list` | Query all entities of a given IFC class |
| `by_id()` | `by_id(id: int) -> entity_instance` | Query by STEP numerical ID |
| `by_guid()` | `by_guid(guid: str) -> entity_instance` | Query by 22-char GlobalId |
| `create_entity()` | `create_entity(type: str, *args, **kwargs) -> entity_instance` | Create a new IFC entity |
| `add()` | `add(inst, _id=None) -> entity_instance` | Add entity (and dependents) from another file |
| `remove()` | `remove(inst) -> None` | Delete entity and nullify references |
| `write()` | `write(path: str, format=None, zipped=False) -> None` | Save to disk (.ifc or .ifcZIP) |
| `traverse()` | `traverse(inst, max_levels=None, breadth_first=False) -> list` | Get all referenced instances recursively |
| `get_inverse()` | `get_inverse(inst) -> set` | Find all entities that reference this entity |

**`ifcopenshell.entity_instance`** — Represents a single IFC entity.

```python
wall = model.by_type("IfcWall")[0]

# Attribute access by name
print(wall.Name)           # "Basic Wall:Generic - 200mm"
print(wall.GlobalId)       # "2XQ$n5SLP5MBLyL442paFx"
print(wall.Description)    # None or string

# Attribute access by index
print(wall[0])             # GlobalId (first attribute)
print(wall[2])             # Name (third attribute)

# Class checking
wall.is_a()                # "IfcWall"
wall.is_a("IfcProduct")   # True (checks inheritance)
wall.is_a("IfcSlab")      # False

# STEP ID
wall.id()                  # 122 (numerical)

# Full info dictionary
info = wall.get_info()     # {"id": 122, "type": "IfcWall", "GlobalId": "...", ...}
info = wall.get_info(recursive=True)  # Expands nested references
```

### 2.3 High-Level API (ifcopenshell.api)

IfcOpenShell provides a structured API organized by domain. This is the RECOMMENDED approach for creating and modifying IFC files (rather than low-level `create_entity` calls).

```python
import ifcopenshell.api.project
import ifcopenshell.api.root
import ifcopenshell.api.unit
import ifcopenshell.api.context
import ifcopenshell.api.spatial
import ifcopenshell.api.aggregate
import ifcopenshell.api.geometry

# Create a complete project from scratch
model = ifcopenshell.api.project.create_file()
project = ifcopenshell.api.root.create_entity(model, ifc_class="IfcProject", name="My Project")
ifcopenshell.api.unit.assign_unit(model)

# Representation context
body = ifcopenshell.api.context.add_context(model, context_type="Model")
body = ifcopenshell.api.context.add_context(
    model, context_type="Model", context_identifier="Body",
    target_view="MODEL_VIEW", parent=body
)

# Spatial hierarchy
site = ifcopenshell.api.root.create_entity(model, ifc_class="IfcSite", name="My Site")
building = ifcopenshell.api.root.create_entity(model, ifc_class="IfcBuilding", name="My Building")
storey = ifcopenshell.api.root.create_entity(model, ifc_class="IfcBuildingStorey", name="Ground Floor")

ifcopenshell.api.aggregate.assign_object(model, relating_object=project, products=[site])
ifcopenshell.api.aggregate.assign_object(model, relating_object=site, products=[building])
ifcopenshell.api.aggregate.assign_object(model, relating_object=building, products=[storey])

# Create a wall with geometry
wall = ifcopenshell.api.root.create_entity(model, ifc_class="IfcWall", name="My Wall")
representation = ifcopenshell.api.geometry.add_wall_representation(
    model, context=body, length=5, height=3, thickness=0.2
)
ifcopenshell.api.geometry.assign_representation(model, product=wall, representation=representation)

# Place in spatial structure
ifcopenshell.api.spatial.assign_container(model, relating_structure=storey, products=[wall])

model.write("output.ifc")
```

### 2.4 Utility Modules

The `ifcopenshell.util` package provides helper functions that abstract common patterns:

**`ifcopenshell.util.element`** — Element queries:
```python
import ifcopenshell.util.element

# Get all properties (occurrence + type combined)
psets = ifcopenshell.util.element.get_psets(wall)
# Returns: {"Pset_WallCommon": {"IsExternal": True, "LoadBearing": False, ...}, ...}

# Properties only (no quantities)
psets = ifcopenshell.util.element.get_psets(wall, psets_only=True)

# Get the type object
wall_type = ifcopenshell.util.element.get_type(wall)

# Get spatial container
container = ifcopenshell.util.element.get_container(wall)  # Returns IfcBuildingStorey

# Get decomposition (children)
elements = ifcopenshell.util.element.get_decomposition(storey)

# Copy operations
shallow_copy = ifcopenshell.util.element.copy(model, wall)
deep_copy = ifcopenshell.util.element.copy_deep(model, wall)
```

**`ifcopenshell.util.placement`** — Coordinate transforms:
```python
import ifcopenshell.util.placement

# Get 4x4 transformation matrix
matrix = ifcopenshell.util.placement.get_local_placement(wall.ObjectPlacement)
# matrix[:,3][:3] gives XYZ coordinates
```

**`ifcopenshell.util.unit`** — Unit handling:
```python
import ifcopenshell.util.unit

# Get conversion factor to SI meters
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
si_meters = ifc_length_value * unit_scale
ifc_value = si_meters / unit_scale
```

**`ifcopenshell.util.classification`** — Classification system access:
```python
import ifcopenshell.util.classification

references = ifcopenshell.util.classification.get_references(wall)
for ref in references:
    system = ifcopenshell.util.classification.get_classification(ref)
```

### 2.5 Geometry Processing

IfcOpenShell includes a geometry engine (`ifcopenshell.geom`) that converts parametric IFC geometry to tessellated meshes:

```python
import ifcopenshell.geom

settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

# Process single element
shape = ifcopenshell.geom.create_shape(settings, wall)
# shape.geometry contains vertices, faces, normals
verts = shape.geometry.verts      # flat array [x1,y1,z1, x2,y2,z2, ...]
faces = shape.geometry.faces      # flat array [i1,i2,i3, ...]

# Iterate all products
iterator = ifcopenshell.geom.iterator(settings, model, multiprocessing.cpu_count())
if iterator.initialize():
    while True:
        shape = iterator.get()
        # process shape...
        if not iterator.next():
            break
```

**Critical note for cross-technology integration**: The geometry engine ALWAYS produces tessellated (triangulated) meshes, regardless of the original IFC geometry type (BREP, extruded solid, CSG, etc.). This means:
- Parametric information is LOST during geometry extraction.
- Round-tripping geometry through tessellation degrades quality.
- Different tessellation settings produce different vertex counts.

---

## 3. web-ifc (ThatOpen Engine)

### 3.1 Overview

web-ifc is a WebAssembly-based IFC parser written in C++ and compiled to WASM via Emscripten. It runs in browsers and Node.js. Current version: **0.0.66+** (rapidly iterating, check npm for latest).

**Verified against**: [ThatOpen/engine_web-ifc GitHub](https://github.com/ThatOpen/engine_web-ifc) source code.

### 3.2 Architecture

- **Language**: TypeScript API wrapper around C++/WASM core
- **Build variants**: `web-ifc.wasm` (browser), `web-ifc-mt.wasm` (multi-threaded browser), `web-ifc-node.wasm` (Node.js)
- **Build requirements**: Emscripten v4.0.23+, CMake v3.18+, Node v16+
- **Codebase**: ~72% TypeScript, ~28% C++

### 3.3 IfcAPI Class

The `IfcAPI` class is the single entry point for all operations:

```javascript
import * as WebIFC from "web-ifc";

const ifcApi = new WebIFC.IfcAPI();
await ifcApi.Init();  // MUST be called before any other method

// Load model from buffer
const data = new Uint8Array(arrayBuffer);
const modelID = ifcApi.OpenModel(data, { COORDINATE_TO_ORIGIN: true });

// Query schema
const schema = ifcApi.GetModelSchema(modelID);  // "IFC4", "IFC2X3", etc.
```

**Core Methods**:

| Method | Signature | Purpose |
|--------|-----------|---------|
| `Init()` | `async Init(locateFile?, forceSingleThread?): Promise<void>` | Initialize WASM module |
| `OpenModel()` | `OpenModel(data: Uint8Array, settings?): number` | Parse IFC, returns modelID |
| `CreateModel()` | `CreateModel(model: NewIfcModel, settings?): number` | Create blank model |
| `CloseModel()` | `CloseModel(modelID: number): void` | Free memory |
| `SaveModel()` | `SaveModel(modelID: number): Uint8Array` | Export to buffer |
| `GetLine()` | `GetLine(modelID, expressID, flatten?, inverse?): IfcLineObject` | Get entity by ID |
| `GetLines()` | `GetLines(modelID, expressIDs[], flatten?, inverse?): IfcLineObject[]` | Batch get |
| `GetAllLines()` | `GetAllLines(modelID): IfcLineObject[]` | Get all entities |
| `GetLineType()` | `GetLineType(modelID, expressID): number` | Get IFC type code |
| `WriteLine()` | `WriteLine(modelID, lineObject): number` | Write/update entity |
| `GetFlatMesh()` | `GetFlatMesh(modelID, expressID): FlatMesh` | Get tessellated geometry |
| `StreamAllMeshes()` | `StreamAllMeshes(modelID, callback): void` | Stream geometry |
| `GetGeometry()` | `GetGeometry(modelID, geometryExpressID): IfcGeometry` | Raw geometry data |
| `GetCoordinationMatrix()` | `GetCoordinationMatrix(modelID): number[]` | Global transform |
| `GetAllTypesOfModel()` | `GetAllTypesOfModel(modelID): IfcType[]` | List all entity types |
| `IsModelOpen()` | `IsModelOpen(modelID): boolean` | Check model state |

### 3.4 Property Access Pattern

```javascript
// Get a wall's properties
const wallLine = ifcApi.GetLine(modelID, wallExpressID, true);
// wallLine.Name.value, wallLine.GlobalId.value, etc.

// Get all walls
const allLines = ifcApi.GetAllLines(modelID);
// Filter by type code:
for (const expressID of allLines) {
    const typeCode = ifcApi.GetLineType(modelID, expressID);
    if (typeCode === WebIFC.IFCWALL) {
        const wall = ifcApi.GetLine(modelID, expressID, true);
        // process wall...
    }
}
```

### 3.5 Key Differences: web-ifc vs IfcOpenShell

| Aspect | IfcOpenShell | web-ifc |
|--------|-------------|---------|
| Language | Python (+ C++ core) | JavaScript/TypeScript (+ C++/WASM) |
| Runtime | Desktop / server | Browser + Node.js |
| Query style | `model.by_type("IfcWall")` | `ifcApi.GetLine(modelID, id)` + type filtering |
| Property access | `wall.Name` (attribute names) | `wallLine.Name.value` (wrapped values) |
| Geometry | `ifcopenshell.geom.create_shape()` | `ifcApi.GetFlatMesh(modelID, id)` |
| High-level API | `ifcopenshell.api.*` (structured) | None — low-level only |
| Utility functions | `ifcopenshell.util.*` (element, placement, unit) | None built-in |
| Schema support | IFC2X3, IFC4, IFC4X3 | IFC2X3, IFC4, IFC4X3 (partial) |
| Property sets | `ifcopenshell.util.element.get_psets()` | Manual traversal of IfcRelDefinesByProperties |
| File creation | Full API support | Basic via `CreateModel()` + `WriteLine()` |
| Multi-threading | Via Python multiprocessing | WASM multi-threaded variant |
| Model ID | Not needed (one file = one object) | Required for all operations |

**Critical integration insight**: Converting between IfcOpenShell and web-ifc property access patterns is a major source of bugs. IfcOpenShell returns Python native types directly (`wall.Name` returns `str`), while web-ifc wraps values in objects (`wallLine.Name.value`). Property set traversal that is one function call in IfcOpenShell (`get_psets()`) requires manual relationship traversal in web-ifc.

---

## 4. FreeCAD IFC Support

### 4.1 NativeIFC Mode

FreeCAD 1.0+ (formerly 0.21+) includes the BIM Workbench with NativeIFC mode, developed by Yorik van Havre. NativeIFC means the IFC file IS the data structure — there is no intermediate translation layer.

**Verified against**: [FreeCAD BIM Workbench wiki](https://wiki.freecad.org/BIM_Workbench) and [NativeIFC GitHub](https://github.com/yorikvanhavre/FreeCAD-NativeIFC).

Key characteristics:
- The IFC file content is rendered directly in FreeCAD.
- Any modification in FreeCAD directly modifies the underlying IFC data.
- No import/export conversion step — the IFC file is always the single source of truth.
- Powered by IfcOpenShell internally (same Python library).
- Funded by NLNet Foundation (since March 2024).

### 4.2 FreeCAD IFC Tools

| Tool | Purpose |
|------|---------|
| IFC Elements Manager | Control which elements export to IFC |
| IFC Properties Manager | Manage property set attachments |
| IFC Quantities Manager | Handle explicit quantity exports |
| IFC Explorer | Pre-import structural analysis |
| IFC Diff | Visual comparison of two IFC files |

### 4.3 Integration Considerations

**FreeCAD internal representation vs IFC**:
- In NativeIFC mode, FreeCAD objects ARE IFC entities (no mapping needed).
- In legacy (non-NativeIFC) mode, FreeCAD Part objects are converted to/from IFC, which introduces data loss at the boundary.
- FreeCAD uses OpenCASCADE (BREP) geometry internally; IFC supports multiple geometry types (extrusion, BREP, tessellation, CSG). Conversion between these representations is lossy.

**Round-trip editing**:
- NativeIFC: lossless round-trip (the IFC file IS the model).
- Legacy mode: lossy round-trip — properties may be dropped, geometry may be tessellated, relationships may be simplified.

**Version compatibility**: FreeCAD's IfcOpenShell integration requires matching versions. In February 2025, updates were made to improve version detection robustness (GitHub PR #19823).

---

## 5. Schema Mapping Between Formats

This section documents how IFC data maps to various target representations. This is the core of cross-technology integration and the primary source of data loss.

### 5.1 IFC → Blender (via Bonsai/BlenderBIM)

| IFC Concept | Blender Representation | Loss? |
|------------|----------------------|-------|
| IfcProduct | Blender Object (bpy.types.Object) | No |
| IfcProduct.Name | Object.name | No |
| Geometry (any type) | Mesh (tessellated triangles) | YES — parametric info lost |
| IfcPropertySet | Custom Properties on Object | No (via Bonsai) |
| IfcRelAggregates | Collection hierarchy | Partial |
| IfcRelContainedInSpatialStructure | Collection membership | Partial |
| IfcBuildingStorey | Empty Object / Collection | Simplified |
| IfcMaterial | Blender Material (simplified) | YES — layer sets simplified |
| IfcRelDefinesByType | Shared data (Bonsai manages) | No (via Bonsai) |
| Multiple representations | Separate mesh data blocks | Accessible but not intuitive |

**Critical note**: Bonsai (formerly BlenderBIM) maintains a parallel IFC data structure alongside Blender's native objects. The Blender mesh is ALWAYS tessellated for display, but the underlying IFC geometry is preserved in Bonsai's internal representation. Without Bonsai, an IFC-to-Blender conversion loses all semantic data.

**Verified against**: [Bonsai extension](https://extensions.blender.org/add-ons/bonsai/) and IfcOpenShell issue tracker.

### 5.2 IFC → Three.js (via web-ifc-three / ThatOpen Components)

| IFC Concept | Three.js Representation | Loss? |
|------------|------------------------|-------|
| IfcProduct | THREE.Object3D / THREE.Mesh | Simplified |
| Geometry | THREE.BufferGeometry (triangulated) | YES — parametric info lost |
| Spatial hierarchy | Object3D parent/child | Partial — flattened for performance |
| IfcPropertySet | Not in scene graph — queried via IfcAPI | Architecture split |
| Materials | THREE.MeshLambertMaterial (basic) | YES — simplified |
| IfcRelationship | Not represented in scene graph | YES — must query separately |
| Large models | Fragment-based streaming | Restructured for GPU efficiency |

**Architecture split**: In modern ThatOpen Components, the geometry lives in Three.js as "Fragments" (optimized binary format built on Google Flatbuffers), while semantic data (properties, relationships) stays in web-ifc and is queried on demand. This means the scene graph does NOT contain IFC properties — you must maintain a parallel data channel.

### 5.3 IFC → web-ifc Flat Arrays

| IFC Concept | web-ifc Representation | Loss? |
|------------|----------------------|-------|
| IfcEntity | IfcLineObject (flat JS object) | No |
| Attributes | Wrapped values (.value property) | No, but different API |
| Relationships | Separate lines, manual traversal | Architecture difference |
| Geometry | FlatMesh (vertex/index arrays) | YES — tessellated |
| Property sets | Lines with expressIDs | Must traverse manually |

### 5.4 IFC → Speckle Base Objects

| IFC Concept | Speckle Representation | Loss? |
|------------|----------------------|-------|
| IfcProduct | DataObject | No |
| IFC Class | `properties["IFC Type"]` | No |
| GlobalId | `properties["IFC GUID"]` + applicationId | No |
| IfcPropertySet | `properties["Property Sets"]` nested | No |
| IFC Attributes | `properties["IFC Attributes"]` nested | No |
| Geometry | `displayValue[]` as Speckle Mesh | YES — tessellated |
| Spatial hierarchy | Collection hierarchy | Preserved |
| Materials | RenderMaterial proxy objects | Simplified |

**Verified against**: [Speckle IFC Schema documentation](https://docs.speckle.systems/developers/data-schema/connectors/ifc-schema).

Speckle uses a flat `properties` dictionary approach — all IFC-specific data lives under `properties` without connector-specific extensions. This makes Speckle a relatively lossless semantic transport (properties and relationships survive), but geometry is always tessellated.

### 5.5 IFC → FreeCAD Part Objects (Legacy Mode)

| IFC Concept | FreeCAD Representation | Loss? |
|------------|----------------------|-------|
| IfcProduct | Part::Feature / Arch object | Mapped |
| Geometry | OpenCASCADE BREP shape | BREP preserved, others tessellated |
| IfcPropertySet | Custom properties | Partial — depends on import settings |
| Spatial hierarchy | Document tree | Simplified |
| Relationships | Not natively represented | YES |

In NativeIFC mode, this mapping does not apply — the IFC data is accessed directly.

### 5.6 Summary: What Survives Cross-Technology Transfer

| Data Type | IfcOpenShell | web-ifc | Blender+Bonsai | Three.js | Speckle | FreeCAD NativeIFC |
|-----------|-------------|---------|----------------|----------|---------|-------------------|
| Entity class | FULL | FULL | FULL | Partial | FULL | FULL |
| GlobalId | FULL | FULL | FULL | Via API | FULL | FULL |
| Properties | FULL | FULL | FULL | Via API | FULL | FULL |
| Quantities | FULL | FULL | FULL | Via API | FULL | FULL |
| Relationships | FULL | FULL | Partial | NONE in scene | Partial | FULL |
| Parametric geometry | FULL | FULL (stored) | Display only | NONE | NONE | FULL |
| Materials | FULL | FULL | Simplified | Simplified | Simplified | FULL |
| Spatial structure | FULL | FULL | Collections | Partial | Collections | FULL |

---

## 6. Common Integration Failures

### 6.1 Property Loss During Conversion

**Scenario**: IFC → JSON/CSV export drops property sets.
**Root cause**: Property sets are linked via `IfcRelDefinesByProperties` — tools that do not traverse relationships will miss them entirely.
**Mitigation**: ALWAYS use `ifcopenshell.util.element.get_psets()` (Python) or manually traverse `IfcRelDefinesByProperties` (web-ifc).

**Scenario**: Type-level properties not included.
**Root cause**: Properties on `IfcWallType` are shared by all `IfcWall` instances but require following `IfcRelDefinesByType` to discover.
**Mitigation**: Query both occurrence AND type: `get_psets(wall)` in IfcOpenShell already includes type-level psets by default.

### 6.2 Geometry Degradation (BREP → Mesh)

**Scenario**: Curved surfaces become faceted after conversion.
**Root cause**: IFC stores parametric geometry (extruded solids, swept solids, BREP). Every visualization tool (Blender, Three.js, web-ifc) tessellates to triangles. Round-tripping tessellated geometry back to IFC produces a `IfcTriangulatedFaceSet` instead of the original parametric representation.
**Impact**: File size increases (10-100x for complex models), curved edges become jagged, boolean operations fail on tessellated geometry.
**Mitigation**: NEVER overwrite original IFC geometry with tessellated versions. Keep the IFC file as the single source of truth.

### 6.3 Relationship Loss

**Scenario**: Spatial containment, aggregation, and type assignments disappear when converting to Three.js scene graph.
**Root cause**: Three.js has a parent-child hierarchy (Object3D), but IFC relationships are many-to-many, objectified entities. There is no direct mapping.
**Impact**: You cannot answer "which storey contains this wall?" from a Three.js scene alone.
**Mitigation**: Maintain a parallel data structure (e.g., a Map<expressID, relationships>) alongside the Three.js scene.

### 6.4 Schema Version Mismatches

**Scenario**: IFC4X3 file opened in IFC4-only tool.
**Root cause**: IFC4X3 adds ~150 new entity types. Tools compiled against IFC4 schema do not recognize them.
**Impact**: Infrastructure entities (IfcAlignment, IfcRoad) silently dropped or cause parse errors.
**Detection**: Check `model.schema` (IfcOpenShell) or `ifcApi.GetModelSchema(modelID)` (web-ifc) before processing.

**Scenario**: IFC2X3 file expected but IFC4 received.
**Root cause**: Many legacy tools still output IFC2X3. Key differences include: `IfcWallStandardCase` (IFC2X3) vs `IfcWall` with `PredefinedType` (IFC4), property set naming differences, removed entities.
**Mitigation**: ALWAYS check schema version first. Use IfcOpenShell's schema-aware API which handles both.

### 6.5 Encoding Issues

**Scenario**: Property values with non-ASCII characters (umlauts, CJK, Cyrillic) appear garbled.
**Root cause**: IFC STEP files use ISO-8859-1 encoding by default, with special escape sequences for extended characters (`\X2\00FC\X0\` for "u"). Not all parsers handle these correctly.
**Impact**: Property values become unreadable, string comparisons fail.
**Detection**: Look for `\X2\` escape sequences in raw IFC text.
**Mitigation**: IfcOpenShell handles decoding automatically. web-ifc also handles it in recent versions. Custom parsers MUST implement IFC string decoding (ISO 10303-21 Part 21).

### 6.6 Large File Handling

**Scenario**: IFC file > 500MB crashes browser or exhausts memory.
**Root cause**: web-ifc loads the entire model into WASM memory. Default WASM memory limit is 2GB.
**Impact**: Out-of-memory crash, browser tab killed.
**Mitigation**:
- web-ifc: Use `StreamAllMeshes()` instead of loading all geometry at once.
- IfcOpenShell: Use the geometry iterator with multiprocessing.
- ThatOpen Components: Fragment-based streaming splits large models into manageable chunks.
- General: Filter by `IfcProduct` subtypes rather than loading everything.

### 6.7 Coordinate System Mismatches

**Scenario**: Model appears at wrong position or rotated when combined with GIS data.
**Root cause**: IFC uses a local coordinate system relative to `IfcSite`. GIS uses global coordinate reference systems (WGS84, UTM). The transformation between them is defined by `IfcMapConversion` (IFC4+) or `IfcSite.RefLatitude/RefLongitude` (IFC2X3).
**Impact**: BIM model placed at (0,0,0) instead of correct geolocation, or offset by thousands of meters.
**Detection**: Check for `IfcMapConversion` entity in the IFC file.
**Mitigation**: Use `ifcApi.GetCoordinationMatrix()` (web-ifc) or parse `IfcMapConversion` manually. For BIM↔GIS integration, ALWAYS establish the coordinate reference system mapping before combining data.

### 6.8 Unit Mismatches

**Scenario**: Dimensions appear 1000x too large or too small.
**Root cause**: IFC files can use any unit system (millimeters, meters, inches, feet). If the consuming tool assumes meters but the file uses millimeters, all dimensions are 1000x off.
**Detection**: `ifcopenshell.util.unit.calculate_unit_scale(model)` returns the conversion factor to SI meters.
**Mitigation**: ALWAYS check units before processing geometry or quantities. Never assume meters.

---

## 7. Technology Version Matrix

| Technology | Current Version | IFC2X3 | IFC4 | IFC4X3 | Notes |
|-----------|----------------|--------|------|--------|-------|
| IfcOpenShell | 0.8.4 | Full | Full | Full | Reference implementation |
| web-ifc | 0.0.66+ | Full | Full | Partial | Geometry for new IFC4X3 types needs verification |
| FreeCAD BIM | 1.0+ | Full | Full | Partial | Via IfcOpenShell |
| Bonsai (BlenderBIM) | 0.8+ | Full | Full | Full | Via IfcOpenShell |
| Speckle IFC | 2.x | Full | Full | Needs verification | Via IFC.js/web-ifc |
| Three.js IFCLoader | r160+ | Full | Full | Partial | Via web-ifc-three |

---

## 8. Verification Notes

The following information was verified against official documentation:
- IFC4 entity hierarchy: verified against IFC4 ADD2 TC1 HTML schema
- IfcOpenShell API methods and signatures: verified against docs.ifcopenshell.org v0.8.4
- web-ifc IfcAPI methods: verified against GitHub source (web-ifc-api.ts)
- FreeCAD NativeIFC: verified against FreeCAD wiki and NativeIFC GitHub repo
- Speckle IFC mapping: verified against docs.speckle.systems

The following items need additional verification:
- Exact IFC4X3 entity count and complete list of new entities (source documents were landing pages, not full schema diffs)
- web-ifc version number (npm publishes frequently; 0.0.66 was latest at time of research but may have advanced)
- Speckle's IFC4X3 support level (documentation does not explicitly state coverage)
- FreeCAD NativeIFC performance characteristics for large files (> 100MB)

---

## Sources

1. [IFC4 ADD2 TC1 Schema](https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/) — buildingSMART International
2. [IFC4X3 Documentation](https://ifc43-docs.standards.buildingsmart.org/) — buildingSMART International
3. [IfcOpenShell 0.8.4 Documentation](https://docs.ifcopenshell.org/) — IfcOpenShell project
4. [IfcOpenShell file class API](https://docs.ifcopenshell.org/autoapi/ifcopenshell/file/index.html)
5. [IfcOpenShell entity_instance API](https://docs.ifcopenshell.org/autoapi/ifcopenshell/entity_instance/index.html)
6. [IfcOpenShell code examples](https://docs.ifcopenshell.org/ifcopenshell-python/code_examples.html)
7. [ThatOpen/engine_web-ifc GitHub](https://github.com/ThatOpen/engine_web-ifc) — web-ifc source code
8. [web-ifc-api.ts source](https://github.com/ThatOpen/engine_web-ifc/blob/main/src/ts/web-ifc-api.ts)
9. [FreeCAD BIM Workbench](https://wiki.freecad.org/BIM_Workbench)
10. [FreeCAD NativeIFC](https://github.com/yorikvanhavre/FreeCAD-NativeIFC)
11. [Speckle IFC Schema](https://docs.speckle.systems/developers/data-schema/connectors/ifc-schema)
12. [Bonsai Blender Extension](https://extensions.blender.org/add-ons/bonsai/)
13. [buildingSMART Forums — IFC4X3 changes](https://forums.buildingsmart.org/t/ifc-4x3-vs-4-1-changes/3217)
