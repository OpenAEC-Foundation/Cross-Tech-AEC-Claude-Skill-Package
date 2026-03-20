# crosstech-errors-conversion — Anti-Patterns

## Anti-Pattern 1: Overwriting IFC Geometry with Tessellated Mesh

**What happens:** Developer exports IFC geometry to triangulated mesh, modifies it, then writes it back to IFC as `IfcTriangulatedFaceSet`.

**Why it fails:**
- The original parametric representation (extruded solid, swept solid, BREP) is permanently lost.
- File size increases 10-100x.
- Curved surfaces become faceted and NEVER recover their smooth definition.
- Boolean operations on tessellated geometry produce artifacts.

**Correct approach:** ALWAYS keep the IFC file as the single source of truth. Modify geometry through IFC-native operations (IfcOpenShell API), NOT through mesh editing.

---

## Anti-Pattern 2: Reading Properties from Direct Attributes Only

**What happens:** Developer reads `element.get_info()` or `ifcApi.GetLine()` and expects to find property sets in the returned data.

**Why it fails:** IFC property sets are NOT direct attributes. They are linked via `IfcRelDefinesByProperties` relationship entities. Direct attribute access returns only the entity's own schema-defined attributes (Name, GlobalId, OwnerHistory, etc.), NOT user-defined properties.

**Correct approach:**
```python
# WRONG
info = element.get_info()  # No property sets here

# CORRECT
psets = ifcopenshell.util.element.get_psets(element)
```

---

## Anti-Pattern 3: Assuming Meters as the Unit

**What happens:** Code processes IFC geometry values directly without checking the model's unit system.

**Why it fails:** IFC files can use any length unit (millimeters, centimeters, meters, inches, feet). Without checking, a model in millimeters renders 1000x too large when interpreted as meters.

**Correct approach:**
```python
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
actual_meters = ifc_value * unit_scale
```

---

## Anti-Pattern 4: Hard-Coding IFC Entity Names Across Schema Versions

**What happens:** Code uses `model.by_type("IfcWallStandardCase")` without checking the schema version.

**Why it fails:** `IfcWallStandardCase` exists in IFC2X3 but was deprecated in IFC4. The query returns an empty list on IFC4 files, silently dropping all walls.

**Correct approach:** ALWAYS check `model.schema` first. Use the broadest applicable type or branch by schema version.

---

## Anti-Pattern 5: Relying on Three.js Scene Graph for IFC Relationships

**What happens:** Developer attempts to determine spatial containment, type assignments, or material links by traversing `Object3D.parent`/`Object3D.children` in Three.js.

**Why it fails:** The Three.js scene graph is optimized for rendering, NOT for IFC semantics. Spatial containment, type assignments, and material relationships are many-to-many objectified relationships that have NO equivalent in a simple parent-child tree. ThatOpen Components further reorganizes geometry into Fragments for GPU performance, breaking any remaining hierarchy.

**Correct approach:** Maintain a parallel data structure (Map, database, or web-ifc queries) for all IFC relationship data. Query semantic relationships through web-ifc, NOT through the scene graph.

---

## Anti-Pattern 6: Loading Entire Large Models into Memory

**What happens:** Code calls `ifcApi.OpenModel()` and then iterates all products, loading all geometry into memory simultaneously.

**Why it fails:** For files >500MB, this exceeds WASM memory limits (browser) or system RAM. The process crashes without a useful error message.

**Correct approach:** Use streaming APIs:
- web-ifc: `StreamAllMeshes()` or filter by type before loading geometry.
- IfcOpenShell: Use `ifcopenshell.geom.iterator()` with multiprocessing.
- ThatOpen Components: Use Fragment-based streaming.

---

## Anti-Pattern 7: Ignoring Schema Version Before Processing

**What happens:** Code processes IFC files without checking whether the file is IFC2X3, IFC4, or IFC4X3.

**Why it fails:**
- IFC4X3 files contain ~150 entity types that IFC4-only tools do not recognize.
- IFC2X3 uses different entity names (`IfcWallStandardCase` vs `IfcWall`).
- Property set names and structures differ between versions.
- Silent data loss occurs when unknown entities are skipped.

**Correct approach:** ALWAYS check schema version as the first step:
```python
schema = model.schema  # "IFC2X3", "IFC4", "IFC4X3"
```

---

## Anti-Pattern 8: Querying Only IfcBuildingElement

**What happens:** Code filters elements with `model.by_type("IfcBuildingElement")` to get "all elements."

**Why it fails:** `IfcBuildingElement` is a subclass of `IfcElement`. It excludes:
- `IfcDistributionElement` (HVAC, plumbing, electrical)
- `IfcFurnishingElement` (furniture)
- `IfcTransportElement` (elevators, escalators)
- `IfcOpeningElement` (voids)
- `IfcCivilElement` (IFC4X3 infrastructure)

**Correct approach:** Use `IfcProduct` (broadest geometric class) or `IfcElement` (all physical elements):
```python
all_physical = model.by_type("IfcElement")  # Includes ALL element subtypes
all_geometric = model.by_type("IfcProduct")  # Includes spatial elements too
```

---

## Anti-Pattern 9: Custom IFC String Parser Without Encoding Support

**What happens:** Developer writes a custom STEP file parser that reads strings as-is without decoding IFC escape sequences.

**Why it fails:** IFC STEP files (ISO 10303-21) use special encoding for non-ASCII characters:
- `\X2\00FC\X0\` encodes Unicode character U+00FC (u with umlaut)
- `\S\` encodes ISO-8859 extended characters
- Without decoding, property values are garbled and string comparisons fail.

**Correct approach:** Use IfcOpenShell or web-ifc, which handle decoding automatically. NEVER write a custom IFC parser unless you implement the full ISO 10303-21 string decoding specification.

---

## Anti-Pattern 10: Sending Large Speckle Commits Without Chunking

**What happens:** A pipeline sends >10,000 IFC objects to Speckle in a single commit operation.

**Why it fails:** Speckle serializes all objects to JSON before sending. Large payloads exceed memory limits or time out during transport. The entire send operation fails with no partial progress.

**Correct approach:** Batch objects into chunks of 1,000-5,000 elements per commit. Use Speckle's built-in batching if available, or implement manual chunking.

---

## Anti-Pattern 11: Skipping Quantity Set Generation Before ERPNext Export

**What happens:** An IFC-to-ERPNext cost pipeline assumes all elements have `IfcElementQuantity` (Qto_) sets.

**Why it fails:** Many IFC files, especially from older authoring tools, do NOT include quantity sets. The pipeline produces empty cost lines with zero quantities.

**Correct approach:** ALWAYS check for quantity sets first. If absent, calculate quantities from geometry:
```python
qtos = ifcopenshell.util.element.get_psets(element, qtos_only=True)
if not qtos:
    # Fall back to geometry-based calculation
    shape = ifcopenshell.geom.create_shape(settings, element)
```

---

## Anti-Pattern 12: Trusting Blender Native IFC Import for Semantic Data

**What happens:** Developer uses Blender's built-in IFC importer (File > Import > IFC) instead of Bonsai and expects full IFC data.

**Why it fails:** Blender's native importer imports geometry and basic names only. It does NOT import:
- Property sets
- Type assignments
- Material layer sets
- Spatial relationships
- Classification references

**Correct approach:** ALWAYS use Bonsai (BlenderBIM) for IFC workflows in Blender. Bonsai maintains a full parallel IFC data model.
