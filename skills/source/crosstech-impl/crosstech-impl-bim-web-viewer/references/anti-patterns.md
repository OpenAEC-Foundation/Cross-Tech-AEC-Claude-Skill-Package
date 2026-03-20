# crosstech-impl-bim-web-viewer — Anti-Patterns

## AP-01: Setting WASM Path After Init

**WRONG**:
```typescript
const ifcApi = new IfcAPI();
await ifcApi.Init();
ifcApi.SetWasmPath("/static/wasm/");  // TOO LATE — WASM already loaded or failed
```

**CORRECT**:
```typescript
const ifcApi = new IfcAPI();
ifcApi.SetWasmPath("/static/wasm/");  // BEFORE Init()
await ifcApi.Init();
```

**Why**: `Init()` fetches and compiles the WASM binary. The path MUST be set before this fetch occurs. Setting it afterward has no effect — the module either loaded from the default path or failed silently.

---

## AP-02: Using LoadAllGeometry for Large Models

**WRONG**:
```typescript
const allMeshes = ifcApi.LoadAllGeometry(modelID);
// Allocates ALL geometry into WASM memory at once — crashes on large models
```

**CORRECT**:
```typescript
ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  // Process one mesh at a time — constant memory usage
  convertAndAddToScene(flatMesh);
});
```

**Why**: `LoadAllGeometry` allocates memory for ALL meshes simultaneously. A 200MB IFC file can produce 1GB+ of tessellated geometry, exceeding the ~2GB WASM memory limit. `StreamAllMeshes` processes incrementally.

---

## AP-03: Mixing Three.js Versions

**WRONG**:
```json
{
  "dependencies": {
    "@thatopen/components": "^3.3.2",
    "three": "^0.170.0"
  }
}
```

If `@thatopen/components` depends on `three@0.164.0`, npm may install BOTH versions. Components use their own Three.js internally, causing type mismatches.

**CORRECT**:
```bash
# Check which version components needs
npm ls three
# Install ONLY that version
npm install three@0.164.0
# Verify single version
npm ls three  # Should show exactly ONE entry
```

**Why**: Three.js does not guarantee backward compatibility between minor versions. Internal class structures change, causing `TypeError: Cannot read properties of undefined` or silent rendering failures when two versions coexist.

---

## AP-04: Manually Rotating web-ifc Output

**WRONG**:
```typescript
// web-ifc output is ALREADY in Y-up
const mesh = convertFromWebIfc(flatMesh);
mesh.rotation.x = -Math.PI / 2;  // DOUBLE rotation — model appears wrong
```

**CORRECT**:
```typescript
const mesh = convertFromWebIfc(flatMesh);
// No rotation needed — web-ifc transforms Z-up to Y-up internally
scene.add(mesh);
```

**When rotation IS needed**: ONLY when receiving coordinates from IfcOpenShell or other Z-up sources that do NOT perform the transformation:

```typescript
// Data from IfcOpenShell server endpoint (Z-up)
const serverGeometry = await fetchFromIfcOpenShell();
serverGeometry.applyMatrix4(new THREE.Matrix4().makeRotationX(-Math.PI / 2));
```

---

## AP-05: Forgetting to Close Models

**WRONG**:
```typescript
async function viewModel(url: string) {
  const data = new Uint8Array(await fetch(url).then(r => r.arrayBuffer()));
  const modelID = ifcApi.OpenModel(data);
  // ... render ...
  // User navigates away — model is NEVER closed
  // WASM memory is NEVER freed
}
```

**CORRECT**:
```typescript
let currentModelID: number | null = null;

async function viewModel(url: string) {
  // Close previous model first
  if (currentModelID !== null) {
    ifcApi.CloseModel(currentModelID);
  }

  const data = new Uint8Array(await fetch(url).then(r => r.arrayBuffer()));
  currentModelID = ifcApi.OpenModel(data);
  // ... render ...
}

// Also close on page unload
window.addEventListener("beforeunload", () => {
  if (currentModelID !== null) {
    ifcApi.CloseModel(currentModelID);
  }
});
```

**Why**: WASM memory is separate from JavaScript garbage collection. Opening multiple models without closing them accumulates WASM memory until the tab crashes with `RangeError: WebAssembly.Memory(): could not allocate memory`.

---

## AP-06: Accessing Property Values Without .value

**WRONG**:
```typescript
const wall = ifcApi.GetLine(modelID, expressID, true);
console.log(wall.Name);
// Output: { value: "Wall-001", type: 1 }  — NOT the string you expected
```

**CORRECT**:
```typescript
const wall = ifcApi.GetLine(modelID, expressID, true);
console.log(wall.Name?.value);
// Output: "Wall-001"
```

**Why**: web-ifc wraps all IFC attribute values in typed containers: `{ value: <actual_value>, type: <ifc_type_id> }`. String properties, numeric properties, and boolean properties ALL use this wrapper. ALWAYS access `.value` to get the actual data. ALWAYS use optional chaining (`?.value`) because attributes can be null in IFC.

---

## AP-07: Not Handling Missing Geometry

**WRONG**:
```typescript
ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  // Assumes every entity has geometry — crashes on IfcSpace, IfcZone, etc.
  const geom = ifcApi.GetGeometry(modelID, flatMesh.expressID);
  const verts = ifcApi.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
  // verts may be empty or geom may be null
});
```

**CORRECT**:
```typescript
ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  const geometries = flatMesh.geometries;
  if (geometries.size() === 0) return;  // Skip entities without geometry

  for (let i = 0; i < geometries.size(); i++) {
    const pg = geometries.get(i);
    const geom = ifcApi.GetGeometry(modelID, pg.geometryExpressID);
    const vertSize = geom.GetVertexDataSize();
    if (vertSize === 0) continue;  // Skip empty geometry

    const verts = ifcApi.GetVertexArray(geom.GetVertexData(), vertSize);
    // Safe to process
  }
});
```

**Why**: Not all IFC entities produce renderable geometry. Spatial elements (IfcSpace, IfcZone), abstract elements, and elements with unsupported geometry representations (complex BREP) may produce empty or null geometry data.

---

## AP-08: Building a Custom Viewer When @thatopen/components Suffices

**WRONG**: Spending weeks implementing navigation, selection, highlighting, clipping, property panels, and storey navigation from scratch using raw Three.js + web-ifc.

**CORRECT**: Use @thatopen/components FIRST. Build custom ONLY when you need:
- Integration into an existing Three.js application that cannot adopt the component system
- Custom rendering pipeline (custom shaders, non-standard materials)
- Features that conflict with the component architecture

**Why**: @thatopen/components provides production-tested implementations of ALL standard BIM viewer features. A custom viewer requires implementing and maintaining: raycasting with expressID mapping, fragment-based rendering, selection state management, visibility toggling, clipping plane UI, and memory management. This is thousands of lines of code that @thatopen/components provides out of the box.

---

## AP-09: Not Copying WASM Files to Build Output

**WRONG** (Vite config):
```typescript
// vite.config.ts — no WASM handling
export default defineConfig({
  // web-ifc.wasm is NOT in dist/ after build
});
```

**CORRECT** (Vite config):
```typescript
import { viteStaticCopy } from "vite-plugin-static-copy";

export default defineConfig({
  plugins: [
    viteStaticCopy({
      targets: [{
        src: "node_modules/web-ifc/*.wasm",
        dest: "static/wasm/",
      }],
    }),
  ],
});
```

Then configure the loader:
```typescript
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: { path: "/static/wasm/", absolute: true },
});
```

**Why**: Bundlers (Webpack, Vite, Rollup) do NOT automatically include `.wasm` files in the build output. The WASM binary must be served as a static asset. Without it, `Init()` fails silently or throws a network error.

---

## AP-10: Ignoring Z-Fighting on BIM Models

**WRONG**:
```typescript
// Default materials — coplanar faces flicker
const material = new THREE.MeshPhongMaterial({ color: 0xcccccc });
```

**CORRECT**:
```typescript
const material = new THREE.MeshPhongMaterial({ color: 0xcccccc });
material.polygonOffset = true;
material.polygonOffsetUnits = 1;
material.polygonOffsetFactor = Math.random();
```

**Why**: BIM models contain many coplanar surfaces (wall face touching slab face, overlapping layers). Without polygon offset, the GPU cannot determine which surface is "in front", causing visible flickering (z-fighting). Randomizing the offset factor ensures each material gets a slightly different depth bias.
