# crosstech-errors-conversion — Real Error Examples

## Example 1: IfcOpenShell — Entity Not Found (Schema Mismatch)

**Error message:**
```
RuntimeError: Entity #4528 not found
```

**Context:** Processing an IFC4X3 infrastructure file with code written for IFC4.

**Root cause:** The entity is an `IfcAlignment` (IFC4X3-only). IfcOpenShell recognizes it, but code filtering by `IfcBuildingElement` skips it entirely, and downstream references to it fail.

**Fix:**
```python
schema = model.schema
if schema == "IFC4X3":
    # Include infrastructure entities
    products = model.by_type("IfcProduct")
else:
    products = model.by_type("IfcBuildingElement")
```

---

## Example 2: web-ifc — GetLine Returns Null

**Error message:**
```
TypeError: Cannot read properties of null (reading 'Name')
```

**Context:** Calling `ifcApi.GetLine(modelID, expressID, true)` returns null for entities that exist in the file.

**Root cause:** The entity type is not registered in web-ifc's type map. This happens with IFC4X3 entities that web-ifc does not yet support, or with rarely-used IFC entities.

**Fix:**
```javascript
const line = ifcApi.GetLine(modelID, expressID, true);
if (!line) {
    console.warn(`Entity #${expressID} not supported by web-ifc`);
    continue; // Skip unsupported entities
}
const name = line.Name?.value ?? "(unnamed)";
```

---

## Example 3: IfcOpenShell — Geometry Extraction Failure

**Error message:**
```
RuntimeError: Failed to create shape for #12345=IfcBooleanClippingResult(...)
```

**Context:** Complex boolean operations with small tolerances fail during tessellation.

**Root cause:** The OpenCASCADE geometry kernel cannot resolve the boolean operation due to tolerance issues or self-intersecting geometry.

**Fix:**
```python
settings = ifcopenshell.geom.settings()
settings.set(settings.DISABLE_BOOLEAN_RESULT, True)  # Skip failed booleans
try:
    shape = ifcopenshell.geom.create_shape(settings, element)
except RuntimeError:
    print(f"Skipping element #{element.id()} — geometry unresolvable")
```

---

## Example 4: Blender — Missing Properties After IFC Import (Without Bonsai)

**Error message:** No error — properties are silently absent.

**Context:** IFC file imported via Blender's built-in IFC importer (not Bonsai). Objects appear but have no custom properties.

**Root cause:** Blender's native IFC importer imports only geometry and basic naming. It does NOT traverse `IfcRelDefinesByProperties` to extract property sets.

**Fix:** Use Bonsai (BlenderBIM) instead of Blender's native importer. Bonsai maintains a full IFC data model alongside Blender objects.

---

## Example 5: Three.js — Properties Not on Mesh Object

**Error message:** No error — `mesh.userData` is empty.

**Context:** Using ThatOpen Components to load IFC in Three.js. Meshes render correctly but have no IFC property data.

**Root cause:** ThatOpen Components deliberately separates geometry (Fragments in Three.js) from semantic data (web-ifc). Properties are NOT stored on `THREE.Mesh` objects.

**Fix:**
```javascript
// Properties must be queried separately via web-ifc
const props = ifcApi.GetLine(modelID, expressID, true);
// Or use ThatOpen's IfcPropertiesProcessor
const propsProcessor = new OBC.IfcPropertiesProcessor(components);
await propsProcessor.process(model);
```

---

## Example 6: Unit Mismatch — Model Appears Microscopic

**Error message:** No error — model renders but is barely visible.

**Context:** IFC file uses millimeters. The consuming tool assumes meters. A 10-meter wall appears as 10mm.

**Diagnostic:**
```python
import ifcopenshell.util.unit
scale = ifcopenshell.util.unit.calculate_unit_scale(model)
print(f"Scale: {scale}")  # 0.001 means millimeters
```

**Fix:**
```python
# Apply scale to all geometry values
actual_length_m = ifc_length_value * unit_scale
```

---

## Example 7: Encoding — Property Values Show Escape Sequences

**Symptom:** Property value shows `Stra\X2\00DF\X0\e` instead of `Strasse`.

**Context:** Reading IFC properties with a custom parser that does not decode ISO 10303-21 string encoding.

**Root cause:** IFC STEP format encodes non-ASCII characters as `\X2\XXXX\X0\` (2-byte Unicode) or `\X4\XXXXXXXX\X0\` (4-byte Unicode).

**Fix:** Use IfcOpenShell or web-ifc, which handle decoding automatically. For custom parsers, implement the decoding:
```python
import re

def decode_ifc_string(s):
    """Decode IFC STEP file string encoding (ISO 10303-21)."""
    # Decode \X2\ ... \X0\ sequences (2-byte Unicode)
    def replace_x2(match):
        hex_str = match.group(1)
        chars = [chr(int(hex_str[i:i+4], 16)) for i in range(0, len(hex_str), 4)]
        return ''.join(chars)
    s = re.sub(r'\\X2\\([0-9A-Fa-f]+)\\X0\\', replace_x2, s)
    # Decode \X\ sequences (single extended byte)
    def replace_x(match):
        return chr(int(match.group(1), 16))
    s = re.sub(r'\\X\\([0-9A-Fa-f]{2})', replace_x, s)
    return s
```

---

## Example 8: Large File — Browser Tab Killed

**Error message:** Browser console shows nothing (tab process killed by OS).

**Context:** Loading a 600MB IFC file via web-ifc in the browser.

**Root cause:** WASM memory limit exceeded. The entire file is loaded into WASM linear memory.

**Fix:**
```javascript
// Option 1: Stream geometry instead of loading all at once
ifcApi.StreamAllMeshes(modelID, (mesh) => {
    processSingleMesh(mesh);
});

// Option 2: Filter by type to reduce memory usage
const wallIDs = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCWALL);
// Process only walls, not all products
```

---

## Example 9: Speckle — Geometry Lost in Transport

**Error message:** No error — objects arrive without `displayValue`.

**Context:** Sending IFC data through Speckle. Objects arrive at the receiving end with properties but no geometry.

**Root cause:** The sending connector did not tessellate geometry before sending. Speckle transports `displayValue` meshes, which must be pre-generated.

**Fix:** Ensure the IFC-to-Speckle converter generates `displayValue` mesh representations:
```python
# The Speckle IFC connector handles this automatically
# If building a custom connector, ALWAYS include displayValue:
from specklepy.objects.geometry import Mesh
speckle_obj.displayValue = [mesh_representation]
```

---

## Example 10: FreeCAD — Shape is Null After IFC Import

**Error message:**
```
<Shape object at 0x...> isNull: True
```

**Context:** FreeCAD legacy IFC import produces objects with null shapes for some elements.

**Root cause:** The element uses an IFC geometry representation that the OpenCASCADE kernel cannot convert (e.g., `IfcAdvancedBrep` with complex trimmed surfaces).

**Fix:**
```python
# Option 1: Use NativeIFC mode (avoids BREP conversion entirely)
# Option 2: In legacy mode, check for null shapes and skip
for obj in FreeCAD.ActiveDocument.Objects:
    if hasattr(obj, 'Shape') and obj.Shape.isNull():
        FreeCAD.Console.PrintWarning(f"Null shape: {obj.Name}\n")
```

---

## Example 11: IFC2X3 vs IFC4 — IfcWallStandardCase Not Found

**Error message:**
```python
model.by_type("IfcWallStandardCase")  # Returns empty list on IFC4 file
```

**Context:** Code written for IFC2X3 files applied to an IFC4 file.

**Root cause:** `IfcWallStandardCase` was deprecated in IFC4. Walls are now `IfcWall` with `PredefinedType=STANDARD`.

**Fix:**
```python
schema = model.schema
if schema == "IFC2X3":
    walls = model.by_type("IfcWallStandardCase") + model.by_type("IfcWall")
else:
    walls = model.by_type("IfcWall")
```

---

## Example 12: Partial Conversion — Distribution Elements Missing

**Error message:** No error — export contains only structural elements.

**Context:** Converting IFC to a schedule. HVAC ducts, pipes, and electrical elements are missing.

**Root cause:** The conversion script queries `model.by_type("IfcBuildingElement")`, which excludes `IfcDistributionElement`, `IfcFurnishingElement`, and other non-building product subtypes.

**Fix:**
```python
# Query the broadest geometric class
all_products = model.by_type("IfcProduct")
# Or explicitly include all element subtypes
all_elements = model.by_type("IfcElement")  # Includes Distribution, Furnishing, etc.
```
