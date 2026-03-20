# Anti-Patterns: IfcOpenShell to web-ifc Conversion Mistakes

## AP-001: Assuming Identical Property Access Syntax

**WRONG** -- treating web-ifc properties like IfcOpenShell:
```javascript
// BROKEN: web-ifc wraps values in {value: ..., type: ...} objects
const wall = api.GetLine(modelID, wallID);
console.log(wall.Name);  // Returns {value: "Wall-001", type: 1} NOT the string
```

**CORRECT** -- unwrap the `.value`:
```javascript
const wall = api.GetLine(modelID, wallID);
console.log(wall.Name.value);  // "Wall-001"
console.log(wall.GlobalId.value);  // "2XQ$n5..."
```

**Why**: IfcOpenShell Python bindings automatically unwrap IFC value types to native Python types. web-ifc preserves the IFC type information in wrapper objects. This difference causes silent bugs when string comparisons or JSON serialization behave unexpectedly.

---

## AP-002: Forgetting to Initialize web-ifc WASM

**WRONG** -- calling API methods before Init():
```javascript
const api = new IfcAPI();
const modelID = api.OpenModel(data);  // FAILS: WASM not loaded
```

**CORRECT** -- ALWAYS await Init() first:
```javascript
const api = new IfcAPI();
await api.Init();  // Loads WASM binary -- MUST complete before any call
const modelID = api.OpenModel(data);
```

**Why**: web-ifc uses a C++ core compiled to WebAssembly. The WASM binary must be downloaded and compiled before any IFC operations are possible. Skipping Init() causes cryptic errors or silent failures.

---

## AP-003: Not Closing web-ifc Models (Memory Leak)

**WRONG** -- letting JavaScript garbage collector handle cleanup:
```javascript
function processModel(data) {
  const api = new IfcAPI();
  await api.Init();
  const modelID = api.OpenModel(data);
  // ... process ...
  // No cleanup -- WASM memory is NEVER freed
}
```

**CORRECT** -- ALWAYS call CloseModel():
```javascript
const api = new IfcAPI();
await api.Init();
const modelID = api.OpenModel(data);
try {
  // ... process ...
} finally {
  api.CloseModel(modelID);  // Frees ALL WASM memory for this model
}
```

**Why**: WebAssembly memory is managed separately from the JavaScript heap. The JavaScript garbage collector does NOT know about WASM allocations. Failing to call CloseModel() causes permanent memory leaks that accumulate until the browser tab runs out of memory or is reloaded.

---

## AP-004: Loading All Geometry at Once for Large Models

**WRONG** -- using LoadAllGeometry on large files:
```javascript
// CRASHES: loads entire model geometry into WASM memory simultaneously
const allMeshes = api.LoadAllGeometry(modelID);  // Out of memory on large models
```

**CORRECT** -- use StreamAllMeshes for memory-efficient processing:
```javascript
api.StreamAllMeshes(modelID, (flatMesh) => {
  // Process one element at a time
  // Previous element's geometry can be freed
});
```

**Why**: web-ifc operates within the ~2GB WebAssembly memory limit. LoadAllGeometry allocates memory for ALL geometry simultaneously. A 200MB IFC file can produce 1GB+ of tessellated geometry, exceeding the WASM limit. StreamAllMeshes processes one element at a time, keeping peak memory low.

---

## AP-005: Mixing Coordinate Systems

**WRONG** -- passing IfcOpenShell coordinates directly to Three.js:
```python
# Server (IfcOpenShell) -- Z-UP
shape = ifcopenshell.geom.create_shape(settings, wall)
verts = shape.geometry.verts  # Z-UP: [x, y, z] where Z is vertical
# Sending these to browser without conversion...
```

```javascript
// Browser -- Three.js expects Y-UP
// Geometry appears rotated 90 degrees -- walls are horizontal
```

**CORRECT** -- convert Z-up to Y-up when transferring from IfcOpenShell to browser:
```python
# Server: convert before sending
def z_up_to_y_up(x, y, z):
    return (x, z, -y)  # Swap Y and Z, negate new Z
```

OR apply rotation in Three.js:
```javascript
model.rotation.x = -Math.PI / 2;
```

**Why**: IFC uses Z-up coordinates (ISO standard). Three.js uses Y-up (WebGL convention). web-ifc handles this internally when `COORDINATE_TO_ORIGIN: true`. But when sending IfcOpenShell geometry to the browser outside of web-ifc, you MUST convert manually.

---

## AP-006: Comparing Tessellation Output Between Libraries

**WRONG** -- expecting identical vertices from both libraries:
```python
# IfcOpenShell produces 156 vertices for a wall
shape = ifcopenshell.geom.create_shape(settings, wall)
print(len(shape.geometry.verts) // 3)  # 156
```

```javascript
// web-ifc produces 128 vertices for the SAME wall
const geom = api.GetGeometry(modelID, pg.geometryExpressID);
const verts = api.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
console.log(verts.length / 6);  // 128
```

**CORRECT** -- treat each library's tessellation as independent. NEVER diff vertex arrays between libraries. Compare semantic data (properties, GlobalId) instead.

**Why**: IfcOpenShell uses Open CASCADE (OCCT) for tessellation; web-ifc uses a custom engine. Different algorithms produce different triangle meshes from the same parametric geometry. Vertex counts, triangle layouts, and normal directions ALWAYS differ.

---

## AP-007: Manual Property Traversal When Utilities Exist

**WRONG** -- manually traversing relationships in IfcOpenShell:
```python
# Unnecessarily complex -- reimplements what get_psets() does
for rel in wall.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        if pset.is_a("IfcPropertySet"):
            for prop in pset.HasProperties:
                print(f"{prop.Name}: {prop.NominalValue.wrappedValue}")
```

**CORRECT** -- use the utility function:
```python
psets = ifcopenshell.util.element.get_psets(wall)
# Automatically merges occurrence + type properties
# Returns clean Python dict
```

**WRONG** -- manual traversal in web-ifc when properties helper exists:
```javascript
// Unnecessarily complex
const wall = api.GetLine(modelID, wallID, true, true);
const rels = wall.IsDefinedBy;
// ... manual traversal of IfcRelDefinesByProperties ...
```

**CORRECT** -- use the properties helper:
```javascript
const psets = await api.properties.getPropertySets(modelID, wallID, true);
```

**Why**: Both libraries provide high-level convenience APIs that handle edge cases (type properties, null references, quantity sets). Manual traversal is error-prone and misses type-level properties.

---

## AP-008: Ignoring the flatten Parameter in web-ifc

**WRONG** -- assuming GetLine returns resolved references:
```javascript
const wall = api.GetLine(modelID, wallID);
// wall.OwnerHistory is a Handle: {expressID: 41, type: 5}
console.log(wall.OwnerHistory.OwningUser);  // UNDEFINED -- not resolved
```

**CORRECT** -- pass `flatten = true` to resolve all references:
```javascript
const wall = api.GetLine(modelID, wallID, true);
// wall.OwnerHistory is now the full IfcOwnerHistory object
console.log(wall.OwnerHistory.OwningUser.ThePerson.GivenName.value);
```

**Why**: By default, `GetLine` returns `Handle` objects (containing only expressID and type code) for nested references. This is a performance optimization -- resolving all references recursively is expensive. Use `flatten = true` only when you need deep access, and `flatten = false` when you only need top-level attributes.

---

## AP-009: Not Checking Schema Version Before Querying IFC4X3 Entities

**WRONG** -- querying infrastructure entities without version check:
```python
# CRASHES on IFC4 files: IfcAlignment does not exist in IFC4
alignments = model.by_type("IfcAlignment")
```

```javascript
// Returns empty set on IFC4 files -- silent failure
const alignmentIDs = api.GetLineIDsWithType(modelID, IFCALIGNMENT);
```

**CORRECT** -- ALWAYS check schema first:
```python
if model.schema == "IFC4X3":
    alignments = model.by_type("IfcAlignment")
```

```javascript
if (api.GetModelSchema(modelID) === 'IFC4X3') {
  const alignmentIDs = api.GetLineIDsWithType(modelID, IFCALIGNMENT);
}
```

**Why**: IFC4X3 introduced infrastructure entities (`IfcAlignment`, `IfcRoad`, `IfcRailway`, `IfcBridge`) that do NOT exist in IFC4 or IFC2X3. Querying these on older schemas causes errors (IfcOpenShell) or empty results (web-ifc).

---

## AP-010: Using web-ifc for IFC Authoring

**WRONG** -- trying to build complex IFC models with web-ifc:
```javascript
// web-ifc writing is BASIC: CreateModel + WriteLine
// No high-level API for spatial structure, geometry creation, property sets
const modelID = api.CreateModel({ schema: Schemas.IFC4 });
api.WriteLine(modelID, lineObject);  // Low-level, error-prone
```

**CORRECT** -- use IfcOpenShell for IFC authoring:
```python
import ifcopenshell.api

model = ifcopenshell.api.project.create_file()
project = ifcopenshell.api.root.create_entity(model, ifc_class="IfcProject", name="My Project")
ifcopenshell.api.unit.assign_unit(model)
# Full structured API for spatial hierarchy, geometry, properties, materials
```

**Why**: IfcOpenShell provides a comprehensive authoring API organized by domain (project, root, unit, context, spatial, aggregate, geometry). web-ifc's writing capabilities are limited to low-level line writing without validation or convenience helpers. Use IfcOpenShell for creating or modifying IFC models, and web-ifc for reading and visualizing them.

---

## AP-011: Reusing IfcAPI Instance Across Multiple Models Without Cleanup

**WRONG** -- opening many models without closing:
```javascript
const api = new IfcAPI();
await api.Init();

for (const file of files) {
  const data = new Uint8Array(await fetch(file).then(r => r.arrayBuffer()));
  const modelID = api.OpenModel(data);
  // Process...
  // NEVER closes -- each model accumulates in WASM memory
}
```

**CORRECT** -- close each model after processing:
```javascript
const api = new IfcAPI();
await api.Init();

for (const file of files) {
  const data = new Uint8Array(await fetch(file).then(r => r.arrayBuffer()));
  const modelID = api.OpenModel(data);
  try {
    // Process...
  } finally {
    api.CloseModel(modelID);
  }
}
```

**Why**: Multiple open models share the same ~2GB WASM memory pool. Without cleanup, memory accumulates rapidly. Three 200MB IFC files can exhaust the entire WASM memory budget.

---

## AP-012: Assuming web-ifc GetVertexArray Format Matches IfcOpenShell

**WRONG** -- treating web-ifc vertex arrays like IfcOpenShell:
```javascript
// web-ifc: 6 floats per vertex (x, y, z, nx, ny, nz)
const verts = api.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
const x = verts[0], y = verts[1], z = verts[2];
const x2 = verts[3]; // WRONG: this is nx (normal X), not the next vertex
```

**CORRECT** -- account for interleaved normals:
```javascript
// web-ifc: stride of 6
const x = verts[0], y = verts[1], z = verts[2];
const nx = verts[3], ny = verts[4], nz = verts[5];
const x2 = verts[6]; // Second vertex starts at index 6
```

```python
# IfcOpenShell: separate arrays for vertices and normals
verts = shape.geometry.verts    # [x1,y1,z1, x2,y2,z2, ...] stride of 3
normals = shape.geometry.normals  # [nx1,ny1,nz1, ...] separate array
```

**Why**: IfcOpenShell stores positions and normals in separate flat arrays with stride 3. web-ifc interleaves positions and normals in a single array with stride 6. Misreading the stride corrupts all geometry data.
