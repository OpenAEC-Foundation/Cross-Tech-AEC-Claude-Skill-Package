---
name: crosstech-core-ifc-schema-bridge
description: >
  Use when mapping IFC entities between different AEC tools: IfcOpenShell (Python),
  web-ifc (WebAssembly), Blender/Bonsai, FreeCAD, Speckle, or Three.js.
  Prevents the common mistake of losing properties, relationships, or geometry
  during IFC data conversion across technology boundaries.
  Covers IFC4/IFC4X3 entity hierarchy, property sets, quantity sets, relationships,
  and how each concept maps to representations in target tools.
  Keywords: IFC, schema, mapping, IfcOpenShell, web-ifc, property sets, quantity sets,
  IfcProduct, IfcRelDefinesByProperties, BIM data conversion, IFC4, IFC4X3.
license: MIT
compatibility: "Designed for Claude Code. Covers IFC4 (ISO 16739-1:2018) and IFC4X3 (ISO 16739-1:2024)."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-core-ifc-schema-bridge

## Quick Reference

### IFC Entity Hierarchy (IFC4 / IFC4X3)

```
IfcRoot (abstract)
├── IfcObjectDefinition (abstract)
│   ├── IfcObject (abstract) — individual occurrences
│   │   ├── IfcProduct — physical/spatial objects with placement + shape
│   │   │   ├── IfcElement (walls, slabs, beams, columns, doors, windows)
│   │   │   ├── IfcSpatialElement (site, building, storey, space)
│   │   │   └── IfcAnnotation, IfcGrid, IfcPort
│   │   ├── IfcProcess, IfcControl, IfcResource, IfcActor, IfcGroup
│   ├── IfcTypeObject → IfcTypeProduct → IfcElementType
│   └── IfcContext → IfcProject
├── IfcRelationship (abstract) — objectified relationships
│   ├── IfcRelAggregates — spatial decomposition
│   ├── IfcRelContainedInSpatialStructure — element containment
│   ├── IfcRelDefinesByProperties — property attachment
│   ├── IfcRelDefinesByType — type assignment
│   ├── IfcRelAssociatesMaterial — material assignment
│   └── IfcRelVoidsElement — opening/void cutting
└── IfcPropertyDefinition → IfcPropertySet, IfcElementQuantity
```

### Critical Warnings

**NEVER** convert IFC to another format without checking the schema version first. IFC4X3 entities are silently dropped by IFC4-only tools.

**NEVER** overwrite original IFC geometry with tessellated versions. Tessellation is a one-way operation that destroys parametric information.

**NEVER** query only occurrence-level properties. ALWAYS check both the occurrence AND its type object (`IfcRelDefinesByType`) for a complete property picture.

**NEVER** assume meters as the IFC unit. ALWAYS check units via `ifcopenshell.util.unit.calculate_unit_scale(model)` or parse `IfcUnitAssignment`.

**NEVER** rely solely on Three.js scene graph data for IFC properties. Properties are NOT stored in the scene graph — they MUST be queried via web-ifc separately.

**NEVER** assume IFC relationships map to parent-child trees. IFC relationships are many-to-many objectified entities. Flattening them to a tree ALWAYS loses information.

---

## Technology Boundary

### Side A: IFC Schema (the universal model)

IFC (Industry Foundation Classes) is the ISO-standardized data model for BIM.

**Schema versions**:

| Version | ISO Standard | Domain |
|---------|-------------|--------|
| IFC4 ADD2 TC1 | ISO 16739-1:2018 | Buildings |
| IFC4X3 (4.3.2) | ISO 16739-1:2024 | Buildings + Infrastructure |

**Core concepts**:
- **Entities**: Typed objects inheriting from `IfcRoot` (GlobalId, Name, Description, OwnerHistory)
- **Properties**: Named value containers (`IfcPropertySet`) attached via `IfcRelDefinesByProperties`
- **Quantities**: Measured values (`IfcElementQuantity`) attached via the same relationship mechanism
- **Relationships**: Objectified links — each relationship is an entity itself with its own GlobalId
- **Types**: Shared definitions (`IfcElementType`) linked to occurrences via `IfcRelDefinesByType`
- **Geometry**: Parametric representations (extruded solids, swept solids, BREP, CSG)

**IFC4X3 additions**: `IfcAlignment`, `IfcRoad`, `IfcRailway`, `IfcBridge`, `IfcMarineFacility`, `IfcFacilityPart`, `IfcCourse`, `IfcPavement`, `IfcBorehole`, `IfcGeomodel`, `IfcLinearElement`. These entities do NOT exist in IFC4.

### Side B: Target Formats (per-tool representation)

| Tool | Version | Internal Representation | Language |
|------|---------|------------------------|----------|
| IfcOpenShell | 0.8.x | `entity_instance` objects in `ifcopenshell.file` | Python |
| web-ifc | 0.0.66+ | `IfcLineObject` flat JS objects keyed by expressID | TypeScript |
| Blender + Bonsai | 4.x/5.x + 0.8.x | `bpy.types.Object` + parallel IFC data store | Python |
| Three.js | r160+ | `THREE.Object3D` / `THREE.Mesh` scene graph | JavaScript |
| Speckle | 2.x | `DataObject` with `properties` dict | Multi-language |
| FreeCAD NativeIFC | 1.0+ | Direct IFC access (no mapping needed) | Python |
| FreeCAD Legacy | < 1.0 | `Part::Feature` / Arch objects | Python |

### The Bridge: Mapping Rules

**Schema mapping summary** (full tables in [references/methods.md](references/methods.md)):

| IFC Concept | IfcOpenShell | web-ifc | Blender+Bonsai | Three.js | Speckle |
|-------------|-------------|---------|----------------|----------|---------|
| Entity class | `entity.is_a()` | `GetLineType()` type code | Object custom prop | Not in scene | `properties["IFC Type"]` |
| GlobalId | `entity.GlobalId` | `line.GlobalId.value` | Object custom prop | Via API only | `applicationId` |
| Properties | `get_psets(entity)` | Manual rel traversal | Custom properties | Via API only | `properties["Property Sets"]` |
| Quantities | `get_psets(entity)` | Manual rel traversal | Custom properties | Via API only | `properties["Property Sets"]` |
| Relationships | `get_inverse()` / util | Manual traversal | Partial (collections) | NOT represented | Partial (hierarchy) |
| Geometry | Parametric (stored) | Parametric (stored) | Tessellated mesh | Tessellated BufferGeometry | Tessellated Mesh |
| Materials | Full IFC material | Full (stored) | Simplified Blender mat | MeshLambertMaterial | RenderMaterial proxy |
| Spatial structure | Full hierarchy | Full (stored) | Collection hierarchy | Partial parent-child | Collection hierarchy |

**Data loss matrix** (what is LOST per target):

| Data Type | IfcOpenShell | web-ifc | Blender+Bonsai | Three.js | Speckle | FreeCAD NativeIFC |
|-----------|:-----------:|:-------:|:--------------:|:--------:|:-------:|:-----------------:|
| Entity class | FULL | FULL | FULL | Partial | FULL | FULL |
| GlobalId | FULL | FULL | FULL | Via API | FULL | FULL |
| Properties | FULL | FULL | FULL | Via API | FULL | FULL |
| Quantities | FULL | FULL | FULL | Via API | FULL | FULL |
| Relationships | FULL | FULL | Partial | **LOST** | Partial | FULL |
| Parametric geometry | FULL | FULL | **LOST** (display) | **LOST** | **LOST** | FULL |
| Material layers | FULL | FULL | **Simplified** | **Simplified** | **Simplified** | FULL |
| Spatial structure | FULL | FULL | Partial | Partial | Partial | FULL |

---

## Critical Rules

1. **ALWAYS** check `model.schema` (IfcOpenShell) or `ifcApi.GetModelSchema(modelID)` (web-ifc) before processing any IFC file.
2. **ALWAYS** query properties from BOTH the occurrence AND its type. Use `ifcopenshell.util.element.get_psets(element)` which does this automatically.
3. **ALWAYS** apply unit conversion before using geometry or quantity values. Use `ifcopenshell.util.unit.calculate_unit_scale(model)`.
4. **ALWAYS** maintain a parallel data structure for IFC semantic data when using Three.js (properties, relationships are NOT in the scene graph).
5. **ALWAYS** keep the original IFC file as the single source of truth for geometry. Tessellated exports are for display only.
6. **NEVER** flatten IFC objectified relationships to a simple tree without documenting what is lost.
7. **NEVER** assume all tools support IFC4X3. Infrastructure entities (`IfcAlignment`, `IfcRoad`) require explicit version checks.
8. **NEVER** compare property values across tools without normalizing string encoding. IFC uses ISO-8859-1 with `\X2\` escape sequences.

---

## Decision Tree

```
START: You have IFC data to use in another tool
│
├── Q1: What is the IFC schema version?
│   ├── IFC4 → All tools support this. Proceed.
│   ├── IFC4X3 → Check target tool support:
│   │   ├── IfcOpenShell 0.8.x → Supported
│   │   ├── web-ifc → Partial (verify geometry for infrastructure entities)
│   │   ├── FreeCAD NativeIFC → Partial (via IfcOpenShell)
│   │   └── Blender+Bonsai → Supported (via IfcOpenShell)
│   └── IFC2X3 → Legacy. Note: IfcWallStandardCase → IfcWall with PredefinedType
│
├── Q2: What data do you need in the target?
│   ├── Geometry only → Tessellation is acceptable. Use geometry engine.
│   ├── Properties + Geometry → MUST maintain parallel data channel (especially Three.js)
│   ├── Full semantic model → Use IfcOpenShell or FreeCAD NativeIFC
│   └── Transport between tools → Use Speckle (preserves properties, tessellates geometry)
│
├── Q3: Is round-trip editing required?
│   ├── YES → Use FreeCAD NativeIFC or Bonsai (keep IFC as source of truth)
│   └── NO → Tessellated export is acceptable for visualization
│
└── Q4: Target runtime?
    ├── Python (desktop/server) → IfcOpenShell
    ├── Browser → web-ifc + Three.js (or ThatOpen Components)
    ├── Blender → Bonsai extension (NEVER raw IFC import without Bonsai)
    ├── FreeCAD → NativeIFC mode (ALWAYS prefer over legacy import)
    └── Multi-tool pipeline → Speckle as transport layer
```

---

## Essential Patterns

### Pattern 1: Property Access Across Tools

The same IFC property (`Pset_WallCommon.IsExternal`) is accessed differently in each tool. See [references/examples.md](references/examples.md) for complete code.

**IfcOpenShell** (one function call):
```python
psets = ifcopenshell.util.element.get_psets(wall)
is_external = psets["Pset_WallCommon"]["IsExternal"]
```

**web-ifc** (manual relationship traversal required):
```javascript
// Must traverse IfcRelDefinesByProperties → IfcPropertySet → IfcPropertySingleValue
// See references/examples.md for the full traversal pattern
```

### Pattern 2: Schema Version Detection

```python
# IfcOpenShell — ALWAYS check before processing
model = ifcopenshell.open("model.ifc")
schema = model.schema  # "IFC2X3", "IFC4", "IFC4X3"
if schema == "IFC4X3":
    alignments = model.by_type("IfcAlignment")  # Only in IFC4X3
```

```javascript
// web-ifc — ALWAYS check before processing
const schema = ifcApi.GetModelSchema(modelID);  // "IFC4", "IFC2X3", etc.
```

### Pattern 3: Relationship Types

IFC relationships are objectified — each relationship is an entity with its own ID. This is the **primary source of data loss** when converting to formats with simple parent-child trees.

| Relationship | Structure | Maps To |
|-------------|-----------|---------|
| `IfcRelAggregates` | Project → Site → Building → Storey | Collection hierarchy (Blender, Speckle) |
| `IfcRelContainedInSpatialStructure` | Storey → [Wall, Slab, ...] | Collection membership |
| `IfcRelDefinesByProperties` | Element → PropertySet | Custom properties (Blender), API query (Three.js) |
| `IfcRelDefinesByType` | Wall → WallType | Shared data reference |
| `IfcRelAssociatesMaterial` | Element → MaterialLayerSetUsage | Simplified material |
| `IfcRelVoidsElement` | OpeningElement → Wall | Boolean modifier (Blender), not represented (Three.js) |

---

## Common Operations

### Reading IFC Across Tools

| Operation | IfcOpenShell | web-ifc |
|-----------|-------------|---------|
| Open file | `ifcopenshell.open("f.ifc")` | `ifcApi.OpenModel(uint8Array)` |
| Get all walls | `model.by_type("IfcWall")` | Filter `GetAllLines()` by type code |
| Get name | `wall.Name` | `line.Name.value` |
| Get GlobalId | `wall.GlobalId` | `line.GlobalId.value` |
| Get properties | `util.element.get_psets(wall)` | Manual traversal (see examples) |
| Get container | `util.element.get_container(wall)` | Manual traversal |
| Get type | `util.element.get_type(wall)` | Manual traversal |
| Get geometry | `geom.create_shape(settings, wall)` | `ifcApi.GetFlatMesh(modelID, id)` |
| Get placement | `util.placement.get_local_placement()` | `GetCoordinationMatrix()` |
| Check schema | `model.schema` | `ifcApi.GetModelSchema(modelID)` |

### IFC4 vs IFC4X3 Entity Mapping

| IFC4 Entity | IFC4X3 Equivalent | Notes |
|------------|-------------------|-------|
| `IfcBuilding` | `IfcBuilding` (subtype of `IfcFacility`) | Unchanged but reclassified |
| (no equivalent) | `IfcRoad` | New in IFC4X3 |
| (no equivalent) | `IfcRailway` | New in IFC4X3 |
| (no equivalent) | `IfcBridge` | New in IFC4X3 |
| (no equivalent) | `IfcAlignment` | New in IFC4X3 — linear referencing |
| `IfcSpatialStructureElement` | `IfcSpatialElement` | Deprecated in favor of generalized hierarchy |
| (no equivalent) | `IfcFacilityPart` | New — generalized spatial decomposition |

---

## Reference Links

- [references/methods.md](references/methods.md) — IFC entity mapping tables for all target formats
- [references/examples.md](references/examples.md) — Working code for reading/writing IFC across tools
- [references/anti-patterns.md](references/anti-patterns.md) — Schema mapping mistakes and data loss patterns

### Official Sources

- https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/ — IFC4 ADD2 TC1 schema
- https://ifc43-docs.standards.buildingsmart.org/ — IFC4X3 documentation
- https://docs.ifcopenshell.org/ — IfcOpenShell 0.8.x API
- https://github.com/ThatOpen/engine_web-ifc — web-ifc source and API
- https://docs.speckle.systems/developers/data-schema/connectors/ifc-schema — Speckle IFC mapping
- https://wiki.freecad.org/BIM_Workbench — FreeCAD BIM/NativeIFC
