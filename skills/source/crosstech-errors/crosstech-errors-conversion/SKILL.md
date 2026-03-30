---
name: crosstech-errors-conversion
description: >
  Use when IFC data conversion fails or produces unexpected results: missing geometry,
  lost properties, broken relationships, or schema version mismatches.
  Provides a symptom-to-root-cause decision tree for diagnosing data conversion
  errors across ALL AEC technology boundaries.
  Covers IfcOpenShell, web-ifc, Blender, FreeCAD, Three.js, Speckle, and ERPNext
  conversion failures with specific error messages and fixes.
  Keywords: IFC conversion error, missing geometry, lost properties, schema mismatch,
  BREP to mesh, property loss, encoding error, IFC import failure, IFC won't import,
  data missing after conversion, geometry broken, IFC export error.
license: MIT
compatibility: "Designed for Claude Code. Covers all AEC technology boundaries."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-errors-conversion

## Quick Reference

| Symptom | Category | Section |
|---------|----------|---------|
| Objects appear but have no geometry | Missing Geometry | [Category 1](#error-category-1-missing-geometry) |
| Curved surfaces are faceted/jagged | Geometry Degradation | [Category 1](#error-category-1-missing-geometry) |
| Property sets are empty or absent | Lost Properties | [Category 2](#error-category-2-lost-properties) |
| Type-level properties not found | Lost Properties | [Category 2](#error-category-2-lost-properties) |
| Spatial containment missing | Broken Relationships | [Category 3](#error-category-3-broken-relationships) |
| Material assignments gone | Broken Relationships | [Category 3](#error-category-3-broken-relationships) |
| `RuntimeError: Entity not found` | Schema Mismatch | [Category 4](#error-category-4-schema-version-mismatch) |
| Garbled property values | Encoding Issues | [Category 5](#error-category-5-encoding-issues) |
| Out of memory / crash | Large File | [Category 6](#error-category-6-large-file--memory) |
| Some elements missing from output | Partial Conversion | [Category 7](#error-category-7-partial-conversion) |
| Dimensions 1000x too large/small | Unit Mismatch | [Category 7](#error-category-7-partial-conversion) |

---

## Diagnostic Decision Tree

```
START: Conversion produced unexpected results
  |
  +-- Is geometry missing or degraded?
  |     YES --> Category 1: Missing Geometry
  |     NO  --+
  |            |
  +-- Are properties/quantities missing?
  |     YES --> Category 2: Lost Properties
  |     NO  --+
  |            |
  +-- Are spatial/type/material relationships gone?
  |     YES --> Category 3: Broken Relationships
  |     NO  --+
  |            |
  +-- Did the parser throw a schema/entity error?
  |     YES --> Category 4: Schema Version Mismatch
  |     NO  --+
  |            |
  +-- Are string values garbled or contain \X2\ sequences?
  |     YES --> Category 5: Encoding Issues
  |     NO  --+
  |            |
  +-- Did the process crash or run out of memory?
  |     YES --> Category 6: Large File / Memory
  |     NO  --+
  |            |
  +-- Are some elements missing or dimensions wrong?
        YES --> Category 7: Partial Conversion
```

---

## Error Category 1: Missing Geometry

### Symptom 1A: Objects exist but have no visible geometry

**Side A (IFC source):** The IFC file contains valid `IfcProduct` entities with `IfcShapeRepresentation`.

**Side B (target tool):** Objects appear in the hierarchy but render as invisible or zero-size.

**Root causes:**
1. **Representation context not requested.** IFC elements have multiple representations (Body, BoundingBox, Axis). The consuming tool MUST request the correct context.
2. **Geometry settings misconfigured.** IfcOpenShell `geom.settings()` or web-ifc `OpenModel()` settings exclude the representation.
3. **Coordinate origin offset.** Objects exist but are placed millions of meters from the viewport center.

**Fix (IfcOpenShell):**
```python
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)
shape = ifcopenshell.geom.create_shape(settings, element)
# Verify: len(shape.geometry.verts) > 0
```

**Fix (web-ifc):**
```javascript
const modelID = ifcApi.OpenModel(data, { COORDINATE_TO_ORIGIN: true });
const mesh = ifcApi.GetFlatMesh(modelID, expressID);
// Verify: mesh.geometries.size() > 0
```

### Symptom 1B: Curved surfaces become faceted/jagged

**Root cause:** ALL visualization tools tessellate IFC parametric geometry (extruded solids, swept solids, BREP) into triangles. This is NOT a bug — it is inherent to the conversion.

**Impact:** File size increases 10-100x. Curved edges become visibly segmented. Boolean operations fail on tessellated geometry.

**Rule:** NEVER overwrite original IFC geometry with tessellated versions. ALWAYS keep the IFC file as the single source of truth for parametric geometry.

**Mitigation (IfcOpenShell):** Increase tessellation precision:
```python
settings = ifcopenshell.geom.settings()
settings.set(settings.DEFLECTION_TOLERANCE, 0.001)  # smaller = finer mesh
```

**Mitigation (web-ifc):** web-ifc tessellation settings are internal. For higher fidelity, use IfcOpenShell server-side and send pre-tessellated geometry to the browser.

### Symptom 1C: FreeCAD BREP import loses CSG operations

**Root cause:** FreeCAD legacy import converts IFC to OpenCASCADE BREP shapes. CSG (Constructive Solid Geometry) operations from the IFC file are evaluated and flattened — the operation tree is lost.

**Fix:** Use FreeCAD NativeIFC mode, which accesses IFC data directly without conversion.

---

## Error Category 2: Lost Properties

### Symptom 2A: Property sets are empty after conversion

**Root cause:** IFC property sets are linked to objects via `IfcRelDefinesByProperties` relationships. Tools that read only entity attributes (not relationships) will NEVER find property sets.

**Fix (IfcOpenShell):**
```python
# CORRECT: traverses relationships automatically
psets = ifcopenshell.util.element.get_psets(element)

# WRONG: only reads direct attributes, misses property sets
info = element.get_info()  # No property sets here
```

**Fix (web-ifc):**
```javascript
// Must manually traverse IfcRelDefinesByProperties
const rels = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCRELDEFINESBYPROPERTIES);
// Then filter by RelatedObjects containing your element's expressID
```

### Symptom 2B: Type-level properties missing

**Root cause:** Properties on `IfcWallType` are shared by all `IfcWall` instances but require traversing `IfcRelDefinesByType` to find. Many export scripts skip type-level properties.

**Fix:** ALWAYS query BOTH occurrence-level AND type-level properties:
```python
# IfcOpenShell: get_psets includes type psets by default
psets = ifcopenshell.util.element.get_psets(wall)  # Includes type psets

# To get type psets separately:
wall_type = ifcopenshell.util.element.get_type(wall)
type_psets = ifcopenshell.util.element.get_psets(wall_type) if wall_type else {}
```

### Symptom 2C: Quantity sets (Qto_) missing in export

**Root cause:** Quantity sets (`IfcElementQuantity`) use a different relationship path than property sets. Some tools handle `IfcPropertySet` but skip `IfcElementQuantity`.

**Fix (IfcOpenShell):**
```python
# Get quantities only
qtos = ifcopenshell.util.element.get_psets(element, qtos_only=True)
```

### Symptom 2D: Properties lost in Speckle round-trip

**Root cause:** Speckle stores IFC properties under `properties["Property Sets"]`. If the receiving connector does not map these back to native property structures, they appear as flat key-value pairs.

**Fix:** ALWAYS check `element.properties` in Speckle for nested IFC data. The property path is: `element.properties["Property Sets"]["Pset_WallCommon"]["IsExternal"]`.

---

## Error Category 3: Broken Relationships

### Symptom 3A: Spatial containment lost (no storey assignment)

**Root cause:** IFC uses objectified relationships (`IfcRelContainedInSpatialStructure`) to assign elements to storeys. Target formats with simple parent-child trees (Three.js `Object3D`, Blender collections) cannot represent many-to-many relationships.

**Fix:** Maintain a parallel lookup structure alongside the scene graph:
```javascript
// Three.js: build a containment map from IFC data
const containmentMap = new Map(); // expressID -> storeyExpressID
```

### Symptom 3B: Material assignments gone after export

**Root cause:** IFC materials are linked via `IfcRelAssociatesMaterial`. Materials can be simple (`IfcMaterial`), layered (`IfcMaterialLayerSetUsage`), or profiled (`IfcMaterialProfileSetUsage`). Most target formats support only simple materials.

**Impact:** Layer thickness, profile shapes, and material constituent ratios are lost.

**Fix (IfcOpenShell):**
```python
import ifcopenshell.util.element
material = ifcopenshell.util.element.get_material(wall)
# Returns IfcMaterial, IfcMaterialLayerSetUsage, etc.
```

### Symptom 3C: Opening/void relationships lost

**Root cause:** `IfcRelVoidsElement` links `IfcOpeningElement` to host walls. `IfcRelFillsElement` links doors/windows to openings. After tessellation, these become merged geometry with no semantic link.

**Rule:** NEVER rely on tessellated geometry to determine opening relationships. ALWAYS query the IFC relationship entities.

---

## Error Category 4: Schema Version Mismatch

### Symptom 4A: `RuntimeError` or unknown entity errors

**Error messages:**
- IfcOpenShell: `RuntimeError: Entity not found` or `IfcAlignment is not valid in IFC4`
- web-ifc: `GetLine returned null` for valid expressIDs

**Root cause:** IFC4X3 adds ~150 new entity types (IfcAlignment, IfcRoad, IfcRailway). Tools compiled against IFC4 do not recognize them.

**Fix:** ALWAYS check schema version before processing:
```python
# IfcOpenShell
schema = model.schema  # "IFC2X3", "IFC4", "IFC4X3"

# web-ifc
const schema = ifcApi.GetModelSchema(modelID);  // "IFC4", "IFC2X3", etc.
```

### Symptom 4B: IFC2X3 vs IFC4 entity name differences

**Key entity changes:**
| IFC2X3 | IFC4 / IFC4X3 |
|--------|---------------|
| `IfcWallStandardCase` | `IfcWall` with `PredefinedType=STANDARD` |
| `IfcBeamStandardCase` | `IfcBeam` with `PredefinedType` |
| `IfcRelAssociatesAppliedValue` | `IfcRelAssociatesConstraint` |
| `IfcCalendarDate` | Removed (use ISO 8601 strings) |
| `IfcDateAndTime` | Removed (use ISO 8601 strings) |

**Rule:** NEVER hard-code entity names without checking the schema version first. Use IfcOpenShell's schema-aware API which normalizes differences.

### Symptom 4C: Property set names differ between schema versions

**Root cause:** Some standard property set names changed between IFC2X3 and IFC4. Custom property sets may use any name.

**Fix:** Match property sets by name prefix (`Pset_`, `Qto_`) rather than exact name when supporting multiple schema versions.

---

## Error Category 5: Encoding Issues

### Symptom 5A: Garbled property values (umlauts, CJK characters)

**Error indicators:** Characters like `Ã¼` instead of `u`, `\X2\00FC\X0\` visible in raw output.

**Root cause:** IFC STEP files (ISO 10303-21) use ISO-8859-1 encoding by default. Extended characters use escape sequences: `\X2\00FC\X0\` encodes `u` (U+00FC). Custom parsers MUST implement IFC string decoding.

**Fix:**
- IfcOpenShell: Handles encoding automatically. No action needed.
- web-ifc: Handles encoding in versions 0.0.50+. Update if using older version.
- Custom parsers: Implement ISO 10303-21 string decoding for `\X\`, `\X2\`, `\X4\`, and `\S\` escape sequences.

### Symptom 5B: File fails to parse with encoding error

**Error messages:** `UnicodeDecodeError`, `Invalid byte sequence`

**Root cause:** The IFC file was saved with UTF-8 encoding instead of ISO-8859-1, or the BOM (byte order mark) confuses the parser.

**Fix:** Strip BOM before parsing. For UTF-8 files, convert to ISO-8859-1 or use a parser that accepts both.

---

## Error Category 6: Large File / Memory

### Symptom 6A: Browser tab crashes on IFC load (web-ifc)

**Error:** `RangeError: WebAssembly.Memory()` or tab killed silently.

**Root cause:** web-ifc loads the entire model into WASM memory. Default WASM limit is 2GB. Files >500MB will likely exceed this.

**Fix:**
```javascript
// Use streaming instead of full load
ifcApi.StreamAllMeshes(modelID, (mesh) => {
  // Process one mesh at a time
});
```

**Alternative:** Use ThatOpen Components Fragment-based streaming for large models.

### Symptom 6B: IfcOpenShell geometry processing hangs

**Root cause:** Single-threaded geometry extraction on files with >100,000 products.

**Fix:**
```python
import multiprocessing
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count()
)
```

### Symptom 6C: Memory exhaustion during Speckle transport

**Root cause:** Speckle serializes all objects to JSON before sending. Large models create massive JSON payloads.

**Fix:** Use Speckle's batch send with chunking. NEVER send >10,000 objects in a single commit without chunking.

---

## Error Category 7: Partial Conversion

### Symptom 7A: Some elements missing from output

**Root causes:**
1. **Filter too restrictive.** The conversion script filters by `IfcBuildingElement` but the model uses subclasses like `IfcFurnishingElement` or `IfcDistributionElement`.
2. **Elements not in spatial structure.** Orphaned elements without `IfcRelContainedInSpatialStructure` are skipped by tools that traverse the spatial tree.
3. **Representation missing.** Elements without geometry representations are skipped by visualization-focused tools.

**Fix:** Query by `IfcProduct` (the broadest geometric class) instead of specific subtypes:
```python
# CORRECT: gets ALL products
products = model.by_type("IfcProduct")

# WRONG: misses distribution, furnishing, and other elements
elements = model.by_type("IfcBuildingElement")
```

### Symptom 7B: Dimensions 1000x too large or too small

**Root cause:** IFC files use configurable unit systems. A file in millimeters opened as if it were in meters produces dimensions 1000x too large.

**Fix:** ALWAYS check and apply the unit scale factor:
```python
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
actual_meters = ifc_value * unit_scale
```

### Symptom 7C: ERPNext cost items missing quantities

**Root cause:** The IFC-to-ERPNext pipeline extracts quantities from `IfcElementQuantity` (Qto_ sets). If the IFC file has no quantity sets, the pipeline produces empty cost lines.

**Fix:** Verify quantity sets exist before running the cost pipeline:
```python
qtos = ifcopenshell.util.element.get_psets(element, qtos_only=True)
if not qtos:
    # Quantities must be calculated from geometry, not from IFC metadata
    shape = ifcopenshell.geom.create_shape(settings, element)
```

---

## Critical Rules

1. **ALWAYS check schema version** before processing any IFC file. Use `model.schema` (IfcOpenShell) or `ifcApi.GetModelSchema()` (web-ifc).
2. **ALWAYS check unit scale** before processing geometry or quantities. NEVER assume meters.
3. **NEVER overwrite original IFC geometry** with tessellated versions. The IFC file is the single source of truth.
4. **NEVER query only direct attributes** for properties. ALWAYS traverse `IfcRelDefinesByProperties` relationships.
5. **ALWAYS query BOTH occurrence AND type** properties when extracting data.
6. **NEVER hard-code IFC entity names** without checking the schema version first.
7. **ALWAYS maintain a parallel data structure** for relationships when converting to formats without objectified relationships (Three.js, simple JSON).
8. **NEVER load entire large models into memory at once.** Use streaming/iterator APIs for files >100MB.

---

## Reference Links

- [references/methods.md](references/methods.md) — Diagnostic commands and API calls per tool
- [references/examples.md](references/examples.md) — Real error messages with solutions
- [references/anti-patterns.md](references/anti-patterns.md) — Conversion mistakes that cause errors

### Official Sources

- https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/
- https://ifc43-docs.standards.buildingsmart.org/
- https://docs.ifcopenshell.org/
- https://github.com/ThatOpen/engine_web-ifc
- https://docs.speckle.systems/developers/data-schema/connectors/ifc-schema
