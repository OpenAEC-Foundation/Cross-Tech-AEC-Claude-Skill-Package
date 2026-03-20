# crosstech-impl-ifc-to-threejs — Methods Reference

## web-ifc IfcAPI — Core Methods

### Initialization

```typescript
import { IfcAPI } from "web-ifc";

const ifcApi = new IfcAPI();

// MUST set WASM path BEFORE Init()
ifcApi.SetWasmPath("/static/wasm/");

// Asynchronous — loads WASM module
await ifcApi.Init();
```

### Model Lifecycle

| Method | Signature | Returns | Purpose |
|--------|-----------|---------|---------|
| `OpenModel` | `(data: Uint8Array, settings?: LoaderSettings) => number` | modelID or -1 | Load IFC from buffer |
| `OpenModelFromCallback` | `(callback: Function) => number` | modelID | Stream-load large files |
| `CreateModel` | `(model: NewIfcModel, settings?: LoaderSettings) => number` | modelID | Create empty IFC model |
| `CloseModel` | `(modelID: number) => void` | void | Free ALL WASM memory for model |
| `IsModelOpen` | `(modelID: number) => boolean` | boolean | Check model state |
| `GetModelSchema` | `(modelID: number) => string` | schema string | "IFC2X3", "IFC4", "IFC4X3" |

### LoaderSettings Interface

```typescript
interface LoaderSettings {
  COORDINATE_TO_ORIGIN?: boolean;    // Center geometry at origin (default: true)
  CIRCLE_SEGMENTS?: number;          // Tessellation detail for circles
  MEMORY_LIMIT?: number;             // Max WASM memory in bytes (default: ~2GB)
  TAPE_SIZE?: number;                // Internal buffer size (default: 64MB)
  LINEWRITER_BUFFER?: number;        // Write buffer size
}
```

### Entity Access

| Method | Signature | Returns | Purpose |
|--------|-----------|---------|---------|
| `GetLine` | `(modelID, expressID, flatten?, inverse?, inversePropKey?) => object` | Entity object | Get entity by EXPRESS ID |
| `GetAllLines` | `(modelID) => Vector<number>` | ID vector | All entity IDs |
| `GetLineIDsWithType` | `(modelID, typeID) => Vector<number>` | ID vector | Entities of specific IFC type |

**`flatten` parameter**: When `false` (default), nested references return as `Handle` objects (`{ expressID: number, type: number }`). When `true`, ALL references are recursively resolved to full objects.

**`inverse` parameter**: When `true`, includes inverse attributes — entities that reference this entity. Required for traversing relationships in reverse.

### Geometry Extraction

| Method | Signature | Returns | Purpose |
|--------|-----------|---------|---------|
| `GetGeometry` | `(modelID, expressID) => IfcGeometry` | Geometry object | Get geometry for one element |
| `GetFlatMesh` | `(modelID, expressID) => FlatMesh` | Flat mesh | Get mesh with all placed geometries |
| `GetVertexArray` | `(ptr, size) => Float32Array` | Vertex data | 6 floats/vertex: x,y,z,nx,ny,nz |
| `GetIndexArray` | `(ptr, size) => Uint32Array` | Index data | Triangle indices |
| `LoadAllGeometry` | `(modelID) => Vector<FlatMesh>` | All meshes | Load ALL geometry (memory-intensive) |
| `StreamAllMeshes` | `(modelID, callback) => void` | void | Stream geometry one-by-one |
| `StreamMeshes` | `(modelID, expressIDs[], callback) => void` | void | Stream specific meshes |

### FlatMesh Structure

```typescript
interface FlatMesh {
  expressID: number;                    // IFC entity this geometry belongs to
  geometries: Vector<PlacedGeometry>;   // One or more placed geometries
}

interface PlacedGeometry {
  color: { x: number; y: number; z: number; w: number }; // RGBA (w = alpha)
  flatTransformation: Float64Array;     // 4x4 matrix (column-major, 16 elements)
  geometryExpressID: number;            // Geometry definition ID
}
```

### Properties Helper

```typescript
const props = ifcApi.properties;

// Element properties
const item = await props.getItemProperties(modelID, expressID);

// Property sets (resolved when flatten=true)
const psets = await props.getPropertySets(modelID, expressID, true);

// Materials
const materials = await props.getMaterialsProperties(modelID, expressID);

// Spatial structure (full hierarchy)
const structure = await props.getSpatialStructure(modelID);
```

### Logging

```typescript
import * as WebIFC from "web-ifc";

ifcApi.SetLogLevel(WebIFC.LogLevel.LOG_LEVEL_OFF);    // Silent
ifcApi.SetLogLevel(WebIFC.LogLevel.LOG_LEVEL_ERROR);  // Errors only
ifcApi.SetLogLevel(WebIFC.LogLevel.LOG_LEVEL_INFO);   // Verbose
```

---

## @thatopen/components — Key Components

### Components + Worlds

```typescript
import * as OBC from "@thatopen/components";

const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer>();
```

### IfcLoader

| Method / Property | Signature | Purpose |
|-------------------|-----------|---------|
| `setup` | `(config: { autoSetWasm: boolean, wasm: { path: string, absolute: boolean } }) => Promise<void>` | Configure WASM location |
| `load` | `(data: Uint8Array, local?: boolean, name?: string, options?: object) => Promise<void>` | Load IFC → Fragments → Scene |

### FragmentsManager

| Method / Property | Signature | Purpose |
|-------------------|-----------|---------|
| `list` | `DataMap<string, FragmentsModel>` | All loaded models |
| `init` | `(workerUrl: string) => void` | Initialize with worker |
| `core.update` | `(force?: boolean) => void` | Force render update |
| `dispose` | `() => void` | Free all fragment memory |

### Highlighter (components-front)

| Method / Property | Type | Purpose |
|-------------------|------|---------|
| `highlight` | `(styleName: string) => Promise<void>` | Highlight by raycasting |
| `highlightByID` | `(styleName: string, idMap: ModelIdMap) => Promise<void>` | Highlight by express IDs |
| `clear` | `(styleName?: string) => Promise<void>` | Clear highlights |
| `multiple` | `"none" \| "shiftKey" \| "ctrlKey"` | Multi-select mode |
| `zoomToSelection` | `boolean` | Auto-zoom on select |
| `selection` | `Record<string, ModelIdMap>` | Current selections |

### Hider

| Method | Signature | Purpose |
|--------|-----------|---------|
| `set` | `(visible: boolean, idMap?: ModelIdMap) => Promise<void>` | Show/hide elements |
| `isolate` | `(idMap: ModelIdMap) => Promise<void>` | Show ONLY specified elements |

### ModelIdMap Type

```typescript
type ModelIdMap = Record<string, Set<number>>;
// Key: model ID string, Value: Set of expressIDs
```

---

## Three.js — Relevant API Surface

### BufferGeometry Construction

```typescript
const geometry = new THREE.BufferGeometry();
geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));
geometry.setAttribute("normal", new THREE.BufferAttribute(normals, 3));
geometry.setIndex(new THREE.BufferAttribute(indices, 1));
geometry.applyMatrix4(transformMatrix);
geometry.computeBoundingSphere(); // Optional but improves raycasting performance
```

### Material for IFC Elements

```typescript
// Basic material from IFC surface color
const material = new THREE.MeshPhongMaterial({
  color: new THREE.Color(r, g, b),
  opacity: alpha,
  transparent: alpha < 1.0,
  side: THREE.DoubleSide,
  polygonOffset: true,           // Z-fighting prevention
  polygonOffsetUnits: 1,
  polygonOffsetFactor: Math.random(),
});

// PBR material for higher quality
const pbrMaterial = new THREE.MeshStandardMaterial({
  color: new THREE.Color(r, g, b),
  roughness: 0.7,
  metalness: 0.1,
  side: THREE.DoubleSide,
});
```

### Raycaster

```typescript
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

// Set from mouse position
raycaster.setFromCamera(mouse, camera);

// Intersect scene
const intersects = raycaster.intersectObjects(scene.children, true);
// intersects[0].object  → THREE.Mesh
// intersects[0].point   → THREE.Vector3 (world position)
// intersects[0].face    → THREE.Face (triangle info)
```

### Matrix4 for IFC Placement

```typescript
// web-ifc provides column-major 4x4 matrix as Float64Array (16 elements)
const matrix = new THREE.Matrix4().fromArray(placedGeometry.flatTransformation);

// Apply to geometry (bakes transform into vertices)
geometry.applyMatrix4(matrix);

// OR apply to mesh (preserves original geometry, transform at render time)
mesh.matrixAutoUpdate = false;
mesh.matrix.copy(matrix);
```

---

## Installation

### @thatopen/components Stack

```bash
npm install @thatopen/components @thatopen/components-front @thatopen/fragments three web-ifc
```

ALWAYS verify single Three.js version:
```bash
npm ls three
# MUST show exactly ONE version
```

### Custom web-ifc Stack

```bash
npm install web-ifc three
npm install -D @types/three  # TypeScript definitions
```

WASM files MUST be copied to the build output directory. For Vite:
```typescript
// vite.config.ts
import { defineConfig } from "vite";
export default defineConfig({
  optimizeDeps: { exclude: ["web-ifc"] },
  server: {
    fs: { allow: [".."] },
  },
});
```
