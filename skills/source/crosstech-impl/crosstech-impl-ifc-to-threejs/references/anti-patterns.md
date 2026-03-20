# crosstech-impl-ifc-to-threejs — Anti-Patterns

## AP-01: Double Axis Rotation

**Wrong**:
```typescript
// web-ifc ALREADY converts Z-up → Y-up internally
const modelID = ifcApi.OpenModel(data);
ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  // ... build geometry ...
  mesh.rotation.x = -Math.PI / 2; // WRONG — applies second rotation
  scene.add(mesh);
});
```

**Why it fails**: web-ifc transforms all geometry from IFC Z-up to Three.js Y-up during extraction. Applying `rotation.x = -Math.PI / 2` rotates the already-correct geometry, producing an upside-down or sideways model.

**Correct**:
```typescript
// No rotation needed — web-ifc output is already Y-up
const mesh = new THREE.Mesh(bufGeom, material);
scene.add(mesh);

// ONLY rotate when using Z-up data from IfcOpenShell or raw IFC coordinates
```

---

## AP-02: Searching the Scene Graph for IFC Properties

**Wrong**:
```typescript
// Trying to find IFC properties in Three.js objects
const wall = scene.getObjectByName("IfcWall_123");
const isExternal = wall.userData.properties?.Pset_WallCommon?.IsExternal;
// userData.properties does NOT exist unless you explicitly put it there
```

**Why it fails**: Three.js scene graph objects contain ONLY geometry and transform data. IFC properties (property sets, quantities, type definitions, relationships) are NOT automatically transferred to the scene graph. They exist in the web-ifc model and MUST be queried via `ifcApi.GetLine()` or `ifcApi.properties.getPropertySets()`.

**Correct**:
```typescript
// Query properties from web-ifc using the expressID
const expressID = mesh.userData.expressID;
const entity = ifcApi.GetLine(modelID, expressID, true);
const psets = await ifcApi.properties.getPropertySets(modelID, expressID, true);
```

---

## AP-03: Using LoadAllGeometry for Large Models

**Wrong**:
```typescript
// Loads ALL geometry into memory at once
const allMeshes = ifcApi.LoadAllGeometry(modelID);
for (let i = 0; i < allMeshes.size(); i++) {
  // Process each mesh
}
```

**Why it fails**: `LoadAllGeometry` allocates WASM memory for every mesh simultaneously. For models over 50MB, this exceeds browser memory limits (typically 2-4GB per tab), causing `RangeError: WebAssembly.Memory(): could not allocate memory` or a silent tab crash.

**Correct**:
```typescript
// Stream geometry one element at a time
ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  // Process each mesh, memory is reusable between callbacks
  const geom = ifcApi.GetGeometry(modelID, flatMesh.expressID);
  // ... convert to Three.js ...
});
```

---

## AP-04: Setting WASM Path After Init()

**Wrong**:
```typescript
const ifcApi = new IfcAPI();
await ifcApi.Init();
ifcApi.SetWasmPath("/static/wasm/"); // TOO LATE — WASM already loaded (or failed)
```

**Why it fails**: `Init()` immediately attempts to load the `.wasm` binary. If the path is not set before `Init()`, web-ifc looks in the default location (typically the page root), fails to find the WASM file, and either throws an error or silently fails.

**Correct**:
```typescript
const ifcApi = new IfcAPI();
ifcApi.SetWasmPath("/static/wasm/"); // BEFORE Init()
await ifcApi.Init();
```

---

## AP-05: Three.js Version Mismatch with @thatopen/components

**Wrong**:
```json
{
  "dependencies": {
    "@thatopen/components": "^3.3.2",
    "three": "^0.170.0"
  }
}
```

**Why it fails**: `@thatopen/components` pins a specific Three.js version internally. Installing a different version creates two copies of Three.js in node_modules. Components create objects with one Three.js version while the application uses another, causing `TypeError: Cannot read properties of undefined` or rendering artifacts.

**Correct**:
```bash
# Check which Three.js version components require
npm ls three
# MUST show exactly ONE version in the tree

# If multiple versions appear, align your package.json
# to match the version required by @thatopen/components
```

---

## AP-06: Forgetting to Close Models (Memory Leak)

**Wrong**:
```typescript
async function loadAndDisplay(url: string): Promise<void> {
  const data = new Uint8Array(await fetch(url).then(r => r.arrayBuffer()));
  const modelID = ifcApi.OpenModel(data);
  ifcApi.StreamAllMeshes(modelID, convertToThreeJs);
  // Model left open — WASM memory never freed
}

// User loads multiple models over time → WASM memory grows until crash
```

**Why it fails**: WASM memory is separate from JavaScript's garbage collector. Opening a model allocates WASM memory that is NEVER freed until `CloseModel()` is called. Each new model adds to total memory, eventually exceeding the ~2GB WASM limit.

**Correct**:
```typescript
async function loadAndDisplay(url: string): Promise<void> {
  const data = new Uint8Array(await fetch(url).then(r => r.arrayBuffer()));
  const modelID = ifcApi.OpenModel(data);
  ifcApi.StreamAllMeshes(modelID, convertToThreeJs);

  // Close model AFTER all geometry is extracted and converted
  // Properties will no longer be accessible after this
  ifcApi.CloseModel(modelID);
}

// If you need property access after rendering, keep model open
// but ALWAYS close when the viewer is destroyed
window.addEventListener("beforeunload", () => {
  ifcApi.CloseModel(modelID);
});
```

---

## AP-07: Ignoring Z-Fighting in BIM Models

**Wrong**:
```typescript
const material = new THREE.MeshPhongMaterial({
  color: new THREE.Color(r, g, b),
  // No polygonOffset — Z-fighting on coplanar surfaces
});
```

**Why it fails**: BIM models have many coplanar surfaces — walls meeting slabs, floor finishes on structural slabs, stacked ceiling layers. Without polygon offset, the GPU cannot determine which surface is "in front," producing flickering (Z-fighting) at every junction.

**Correct**:
```typescript
const material = new THREE.MeshPhongMaterial({
  color: new THREE.Color(r, g, b),
  polygonOffset: true,
  polygonOffsetUnits: 1,
  polygonOffsetFactor: Math.random(), // Randomize to separate coplanar faces
});
```

---

## AP-08: Not Storing expressID on Meshes

**Wrong**:
```typescript
ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  // ... build geometry ...
  const mesh = new THREE.Mesh(bufGeom, material);
  scene.add(mesh); // No expressID stored — selection is impossible
});
```

**Why it fails**: Without the expressID stored on the Three.js mesh, raycasting hits return a mesh with no link back to the IFC entity. Property lookup, highlighting, and all semantic operations require the expressID.

**Correct**:
```typescript
ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  // ... build geometry ...
  const mesh = new THREE.Mesh(bufGeom, material);
  mesh.userData.expressID = flatMesh.expressID; // ALWAYS store the mapping
  scene.add(mesh);
});
```

---

## AP-09: Treating Fragments as IFC

**Wrong**:
```typescript
// Trying to use web-ifc API on a Fragments model
const model = await ifcLoader.load(data, false, "building");
const entity = ifcApi.GetLine(0, someExpressID); // WRONG — Fragments is not an IfcAPI model
```

**Why it fails**: @thatopen/components converts IFC to Fragments format during loading. The original web-ifc model is closed after conversion. You cannot use raw `IfcAPI` methods on Fragment data. Use the components API instead.

**Correct**:
```typescript
// Use @thatopen/components API for property access on Fragment models
const fragments = components.get(OBC.FragmentsManager);
// Access model data through the Fragment model's built-in methods
const categories = model.getItemsOfCategories(["IFCWALL"]);
```

---

## AP-10: Losing Real-World Coordinates

**Wrong**:
```typescript
const modelID = ifcApi.OpenModel(data); // Default: COORDINATE_TO_ORIGIN = true
// All geometry is now centered at (0,0,0)
// Original geo-coordinates (easting/northing) are LOST
```

**Why it fails**: `COORDINATE_TO_ORIGIN: true` (the default) subtracts the model's coordinate offset to center geometry at the origin. This makes viewing easier but destroys the real-world position. If you later need to overlay the model on a map or align with other georeferenced models, the coordinates are gone.

**Correct**:
```typescript
// When georeferenced positioning is required
const modelID = ifcApi.OpenModel(data, {
  COORDINATE_TO_ORIGIN: false,
});

// Store the coordination matrix for later use
const coordMatrix = ifcApi.GetCoordinationMatrix(modelID);
```

---

## AP-11: Accessing Properties After CloseModel

**Wrong**:
```typescript
const modelID = ifcApi.OpenModel(data);
ifcApi.StreamAllMeshes(modelID, convertToThreeJs);
ifcApi.CloseModel(modelID); // Frees all data

// Later, on user click:
const entity = ifcApi.GetLine(modelID, expressID); // CRASH — model is closed
```

**Why it fails**: `CloseModel` frees ALL WASM memory for that model, including entity data, properties, and spatial structure. Any subsequent `GetLine`, `getPropertySets`, or other data access calls fail.

**Correct**: Either keep the model open for the lifetime of the viewer, or pre-extract all needed properties into a JavaScript Map before closing:

```typescript
// Option A: Keep model open (recommended for interactive viewers)
// Close only when viewer is destroyed

// Option B: Pre-extract properties (for memory-constrained scenarios)
const propertyCache = new Map<number, any>();
for (const [expressID] of expressIdToMesh) {
  const entity = ifcApi.GetLine(modelID, expressID, true);
  const psets = await ifcApi.properties.getPropertySets(modelID, expressID, true);
  propertyCache.set(expressID, { entity, psets });
}
ifcApi.CloseModel(modelID); // Safe — all data is in JS memory now
```
