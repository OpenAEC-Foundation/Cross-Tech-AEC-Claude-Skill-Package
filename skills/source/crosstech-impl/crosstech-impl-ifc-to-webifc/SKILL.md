---
name: crosstech-impl-ifc-to-webifc
description: >
  Use when converting IFC processing between IfcOpenShell (Python, server-side)
  and web-ifc (WebAssembly, browser-side). Prevents the common mistake of assuming
  identical property access patterns or geometry processing between the two libraries.
  Covers IfcOpenShell 0.8.x vs web-ifc 0.0.77 API differences, property access,
  geometry extraction, streaming large models, and hybrid architecture patterns.
  Keywords: IfcOpenShell, web-ifc, WebAssembly, IFC parser, property access,
  geometry extraction, server-side, browser-side, WASM, IFC conversion.
license: MIT
compatibility: "Designed for Claude Code. Covers IfcOpenShell 0.8.x (Python) and web-ifc 0.0.77 (JavaScript/WASM)."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-impl-ifc-to-webifc

## Quick Reference

### API Mapping Table

| Operation | IfcOpenShell (Python) | web-ifc (JavaScript/WASM) |
|-----------|----------------------|---------------------------|
| Initialize | `import ifcopenshell` | `const api = new IfcAPI(); await api.Init();` |
| Open file | `model = ifcopenshell.open("f.ifc")` | `modelID = api.OpenModel(uint8Array)` |
| Close file | `del model` / garbage collected | `api.CloseModel(modelID)` **MUST call** |
| Get schema | `model.schema` | `api.GetModelSchema(modelID)` |
| Get by type | `model.by_type("IfcWall")` | `api.GetLineIDsWithType(modelID, IFCWALL)` |
| Get by ID | `model.by_id(122)` | `api.GetLine(modelID, 122)` |
| Get name | `wall.Name` | `wall.Name.value` |
| Get GlobalId | `wall.GlobalId` | `wall.GlobalId.value` |
| Get properties | `util.element.get_psets(wall)` | `api.properties.getPropertySets(modelID, id, true)` |
| Get container | `util.element.get_container(wall)` | Manual traversal required |
| Get type | `util.element.get_type(wall)` | Manual traversal required |
| Get geometry | `geom.create_shape(settings, wall)` | `api.GetFlatMesh(modelID, id)` |
| Stream geometry | `geom.iterator(settings, model)` | `api.StreamAllMeshes(modelID, cb)` |
| Create model | `ifcopenshell.file(schema="IFC4")` | `api.CreateModel({schema: Schemas.IFC4})` |
| Write file | `model.write("out.ifc")` | `api.SaveModel(modelID)` returns `Uint8Array` |

### Critical Warnings

**NEVER** assume property access patterns are identical. IfcOpenShell returns Python objects with direct attribute access; web-ifc returns JavaScript objects where values are wrapped in `{value: ...}` containers.

**NEVER** skip `await ifcApi.Init()` in web-ifc. ALL subsequent calls fail silently or throw without WASM initialization.

**NEVER** forget to call `ifcApi.CloseModel(modelID)` in web-ifc. WASM memory is NOT garbage collected by the JavaScript engine.

**NEVER** assume geometry output is identical between the two libraries. IfcOpenShell uses Open CASCADE (OCCT) for tessellation; web-ifc uses a custom engine. Vertex counts and triangle layouts ALWAYS differ.

**NEVER** assume coordinate systems match. IfcOpenShell outputs Z-up coordinates (IFC standard); web-ifc outputs Y-up coordinates (Three.js convention) when `COORDINATE_TO_ORIGIN` is enabled.

**ALWAYS** check the IFC schema version before processing on BOTH sides. Use `model.schema` (IfcOpenShell) and `api.GetModelSchema(modelID)` (web-ifc).

---

## Technology Boundary

### Side A: IfcOpenShell (Python, server-side)

- **Version**: 0.8.x (2026)
- **Language**: C++ core with Python bindings
- **Geometry engine**: Open CASCADE Technology (OCCT) -- robust BREP, boolean operations
- **Runtime**: CPython on server/desktop, no browser support
- **Memory**: Limited by system RAM (no artificial cap)
- **License**: LGPL-3.0
- **Strengths**: Full IFC authoring, complex geometry, batch processing, rich utility library
- **Property access**: Direct attribute access (`wall.Name`), high-level utilities (`get_psets()`)
- **Coordinate system**: Z-up (native IFC convention)

### Side B: web-ifc (JavaScript/WASM, browser-side)

- **Version**: 0.0.77 (2026)
- **Language**: C++ compiled to WebAssembly via Emscripten, TypeScript API wrapper
- **Geometry engine**: Custom engine optimized for speed over robustness
- **Runtime**: Browser (WASM) or Node.js
- **Memory**: Limited to ~2GB (WebAssembly specification limit)
- **License**: MPL-2.0
- **Strengths**: Browser-native, streaming geometry, interactive viewing, offline capable
- **Property access**: `GetLine()` returns objects with `.value` wrappers, `properties` helper for convenience
- **Coordinate system**: Y-up (when `COORDINATE_TO_ORIGIN: true`, default)

### The Bridge: Same IFC file, different access patterns

Both libraries read the SAME IFC file format (STEP Physical File). The bridge is the IFC file itself. Key differences at the boundary:

| Aspect | IfcOpenShell | web-ifc |
|--------|-------------|---------|
| File input | File path string | `Uint8Array` buffer |
| Entity identity | `entity_instance` with `.id()` | EXPRESS ID integer |
| Attribute values | Direct Python types | Wrapped in `{value: ..., type: ...}` |
| Relationship traversal | `get_inverse()` + utility functions | `GetLine(id, flat, inverse)` params |
| Geometry output | Flat arrays (verts, faces) via OCCT | `FlatMesh` with placed geometries |
| Coordinate convention | Z-up (IFC native) | Y-up (Three.js convention) |
| Memory lifecycle | Python garbage collection | Manual `CloseModel()` required |

---

## Critical Rules

1. **ALWAYS** normalize property values when comparing between the two libraries. IfcOpenShell returns `wall.Name` as a plain string; web-ifc returns `wall.Name.value`.
2. **ALWAYS** handle coordinate system differences. When transferring geometry from IfcOpenShell to a web-ifc/Three.js context, rotate -90 degrees around X to convert Z-up to Y-up.
3. **ALWAYS** call `ifcApi.CloseModel(modelID)` when finished with a model in web-ifc. WASM memory leaks are permanent until page reload.
4. **ALWAYS** use `StreamAllMeshes` instead of `LoadAllGeometry` for large models in web-ifc. Loading all geometry at once causes out-of-memory crashes.
5. **ALWAYS** use `ifcopenshell.util.element.get_psets()` on the IfcOpenShell side -- it merges occurrence AND type properties automatically. web-ifc `properties.getPropertySets()` does the same.
6. **NEVER** compare tessellated vertex data between the two libraries. Different geometry engines produce different tessellations of the same parametric geometry.
7. **NEVER** assume IFC4X3 infrastructure entities work identically in both. IfcOpenShell has partial IFC4X3 support; web-ifc support is evolving.
8. **NEVER** pass IfcOpenShell geometry coordinates directly to Three.js without Y-up conversion.

---

## Decision Tree

```
START: You need IFC processing and have both libraries available
|
+-- Q1: Where does the code run?
|   +-- Browser only --> Use web-ifc
|   +-- Server only --> Use IfcOpenShell
|   +-- Both (hybrid app) --> Continue to Q2
|
+-- Q2: What operations are needed?
|   +-- Visualization / interactive viewing --> web-ifc (browser)
|   +-- IFC authoring / model creation --> IfcOpenShell (server)
|   +-- Complex boolean geometry --> IfcOpenShell (OCCT engine)
|   +-- Property inspection --> Either (web-ifc has convenience API)
|   +-- Batch processing (>10 files) --> IfcOpenShell (no memory cap)
|   +-- Model >500MB --> IfcOpenShell (web-ifc hits 2GB WASM limit)
|
+-- Q3: Do you need to transfer data between both?
|   +-- YES --> Define a serialization format (JSON properties, glTF geometry)
|   |   +-- Properties: JSON via REST API
|   |   +-- Geometry: Pre-tessellate on server, send as glTF or Fragments
|   |   +-- Full model: Serve the .ifc file, parse on both sides independently
|   +-- NO --> Use each library independently on its platform
|
+-- Q4: Coordinate system?
    +-- Server sends coordinates to browser --> ALWAYS convert Z-up to Y-up
    +-- Browser sends coordinates to server --> ALWAYS convert Y-up to Z-up
    +-- Each side processes independently --> No conversion needed
```

---

## Essential Patterns

### Pattern 1: Property Access Comparison

**IfcOpenShell** -- one function, all properties:
```python
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# All properties (occurrence + type merged)
psets = ifcopenshell.util.element.get_psets(wall)
# Returns: {"Pset_WallCommon": {"IsExternal": True, ...}, "Qto_WallBaseQuantities": {...}}

is_external = psets["Pset_WallCommon"]["IsExternal"]  # Direct Python bool
```

**web-ifc** -- convenience API available:
```javascript
import { IfcAPI } from 'web-ifc';

const api = new IfcAPI();
await api.Init();
const modelID = api.OpenModel(data);

const wallIDs = api.GetLineIDsWithType(modelID, WebIFC.IFCWALL);
const wallID = wallIDs.get(0);

// Convenience API (recommended)
const psets = await api.properties.getPropertySets(modelID, wallID, true);

// Manual approach (for custom traversal)
const wall = api.GetLine(modelID, wallID, true);
// wall.Name.value --> string
// wall.GlobalId.value --> string
```

### Pattern 2: Geometry Extraction Comparison

**IfcOpenShell** -- single element:
```python
import ifcopenshell.geom

settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

shape = ifcopenshell.geom.create_shape(settings, wall)
verts = shape.geometry.verts   # [x1,y1,z1, x2,y2,z2, ...] Z-UP
faces = shape.geometry.faces   # [i1,i2,i3, ...]
```

**web-ifc** -- single element:
```javascript
const flatMesh = api.GetFlatMesh(modelID, expressID);
const placedGeoms = flatMesh.geometries;

for (let i = 0; i < placedGeoms.size(); i++) {
  const pg = placedGeoms.get(i);
  const geom = api.GetGeometry(modelID, pg.geometryExpressID);
  const verts = api.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
  // verts: [x,y,z,nx,ny,nz, ...] 6 floats per vertex, Y-UP
  const indices = api.GetIndexArray(geom.GetIndexData(), geom.GetIndexDataSize());
  const transform = pg.flatTransformation; // 4x4 matrix as flat array
}
```

### Pattern 3: Streaming Large Models

**IfcOpenShell** -- iterator pattern:
```python
import multiprocessing
import ifcopenshell.geom

settings = ifcopenshell.geom.settings()
iterator = ifcopenshell.geom.iterator(settings, model, multiprocessing.cpu_count())
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)
        # Process shape.geometry.verts, shape.geometry.faces
        if not iterator.next():
            break
```

**web-ifc** -- callback streaming:
```javascript
api.StreamAllMeshes(modelID, (flatMesh) => {
  const expressID = flatMesh.expressID;
  // Process geometry per element -- memory-efficient
  // Each callback processes one element, then memory can be freed
});
```

### Pattern 4: Hybrid Architecture (Server + Browser)

```
Browser (web-ifc)                    Server (IfcOpenShell)
+-------------------+               +----------------------+
| web-ifc loads .ifc|               | IfcOpenShell loads   |
| for visualization |  <-- REST --> | same .ifc for:       |
|                   |    API/WS     | - Validation         |
| StreamAllMeshes   |               | - Complex queries    |
| for 3D rendering  |               | - Model authoring    |
|                   |               | - >2GB files         |
| properties API    |               | - Batch processing   |
| for inspection    |               | - Format conversion  |
+-------------------+               +----------------------+
```

**Server pre-processing** (recommended for large models):
```python
# Server: convert IFC to Fragments format for instant browser loading
# Use IfcOpenShell for validation, then serve .ifc or pre-converted .frag
import ifcopenshell

model = ifcopenshell.open("large_model.ifc")
errors = []  # Validate on server
for wall in model.by_type("IfcWall"):
    psets = ifcopenshell.util.element.get_psets(wall)
    if "Pset_WallCommon" not in psets:
        errors.append(f"Wall {wall.GlobalId} missing Pset_WallCommon")

# Serve validated file to browser for web-ifc visualization
```

### Pattern 5: Coordinate System Conversion

```python
# Server-side: IfcOpenShell Z-up to Three.js/web-ifc Y-up
import numpy as np

# Rotation matrix: -90 degrees around X axis
z_up_to_y_up = np.array([
    [1,  0,  0, 0],
    [0,  0, -1, 0],
    [0,  1,  0, 0],
    [0,  0,  0, 1]
], dtype=float)

# Apply to IfcOpenShell vertex (x, y_ifc, z_ifc)
# Result: (x, z_ifc, -y_ifc) in Y-up
def convert_z_to_y_up(x, y, z):
    return (x, z, -y)
```

```javascript
// Browser-side: if receiving Z-up coordinates from server
// Apply rotation to Three.js object
model.rotation.x = -Math.PI / 2;
```

**IMPORTANT**: web-ifc handles this internally when `COORDINATE_TO_ORIGIN: true` (default). You ONLY need manual conversion when sending coordinates from IfcOpenShell to the browser outside of web-ifc.

---

## Common Operations

### Reading the Same Property from Both Sides

| Property | IfcOpenShell | web-ifc |
|----------|-------------|---------|
| Name | `wall.Name` (str or None) | `wall.Name?.value` (str or undefined) |
| GlobalId | `wall.GlobalId` (str) | `wall.GlobalId.value` (str) |
| Predefined type | `wall.PredefinedType` (str) | `wall.PredefinedType?.value` (str) |
| Is external | `get_psets(wall)["Pset_WallCommon"]["IsExternal"]` | Via `properties.getPropertySets()` |
| Container | `util.element.get_container(wall)` | Manual: traverse `IfcRelContainedInSpatialStructure` |
| Type object | `util.element.get_type(wall)` | Manual: traverse `IfcRelDefinesByType` |

### Memory Management Comparison

| Concern | IfcOpenShell | web-ifc |
|---------|-------------|---------|
| Max memory | System RAM | ~2GB (WASM limit) |
| Cleanup | Python GC (automatic) | `CloseModel()` **MUST call manually** |
| Large file strategy | Iterator with multiprocessing | `StreamAllMeshes` callback |
| Multiple models | No limit (RAM permitting) | Multiple modelIDs, shared 2GB pool |
| Geometry caching | Managed by OCCT | Manual -- discard after processing |

### Error Handling Differences

| Error | IfcOpenShell | web-ifc |
|-------|-------------|---------|
| File not found | `FileNotFoundError` | Must handle before `OpenModel` (needs `Uint8Array`) |
| Invalid IFC | `RuntimeError` on parse | `OpenModel` returns -1 or throws |
| Unknown entity type | `RuntimeError` | Silent skip or null return |
| Out of memory | OS kills process | WASM memory limit error |

---

## Reference Links

- [references/methods.md](references/methods.md) -- IfcOpenShell API vs web-ifc IfcAPI comparison
- [references/examples.md](references/examples.md) -- Working code for both sides
- [references/anti-patterns.md](references/anti-patterns.md) -- Conversion mistakes

### Official Sources

- https://docs.ifcopenshell.org/ -- IfcOpenShell 0.8.x API documentation
- https://github.com/ThatOpen/engine_web-ifc -- web-ifc source and API
- https://github.com/ThatOpen/engine_web-ifc/blob/main/src/ts/web-ifc-api.ts -- IfcAPI TypeScript source
- https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/ -- IFC4 schema reference
- https://ifc43-docs.standards.buildingsmart.org/ -- IFC4X3 documentation
