# crosstech-impl-bim-web-viewer — Methods Reference

## @thatopen/components Core Classes

### OBC.Components

The root container for all components. Singleton pattern — create once per application.

```typescript
const components = new OBC.Components();
```

### OBC.Worlds

Factory for World objects (Scene + Camera + Renderer groups).

```typescript
const worlds = components.get(OBC.Worlds);
const world = worlds.create<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer>();
```

### OBC.IfcLoader

Converts IFC files to Fragments and loads them into the scene.

| Method | Signature | Purpose |
|--------|-----------|---------|
| `setup` | `(config: IfcLoaderConfig) => Promise<void>` | Configure WASM path and options |
| `load` | `(data: Uint8Array, coordinate?: boolean, name?: string, config?: object) => Promise<FragmentsGroup>` | Load IFC from buffer |

#### IfcLoaderConfig

```typescript
interface IfcLoaderConfig {
  autoSetWasm?: boolean;           // Auto-detect WASM path (default: true)
  wasm?: {
    path: string;                  // Directory containing web-ifc.wasm
    absolute: boolean;             // true = absolute URL, false = relative
  };
}
```

#### Load Config — processData

```typescript
await ifcLoader.load(data, false, "model-name", {
  processData: {
    progressCallback: (progress: number) => void,
  },
});
```

### OBC.FragmentsManager

Central manager for all loaded Fragment models.

| Method / Property | Type | Purpose |
|-------------------|------|---------|
| `list` | `DataMap<string, FragmentsGroup>` | All loaded models |
| `list.onItemSet` | `Event` | Fires when a model is loaded |
| `init` | `(workerUrl: string) => void` | Initialize with worker for off-thread processing |
| `core.update` | `(force: boolean) => void` | Force update all model rendering |

#### FragmentsGroup (Model)

```typescript
// Access after loading
const model: FragmentsGroup;

// Key methods
model.useCamera(camera: THREE.Camera);         // Bind camera for LOD
model.object;                                    // THREE.Group — add to scene
model.getItemsOfCategories(cats: string[]);     // Get IDs by IFC type
model.modelId;                                   // Unique model identifier
await model.getBuffer(includeProperties: boolean); // Export to .frag
```

### OBCF.Highlighter

Selection and highlighting via raycasting.

| Method / Property | Signature | Purpose |
|-------------------|-----------|---------|
| `highlight` | `(styleName: string) => Promise<ModelIdMap>` | Highlight via mouse raycast |
| `highlightByID` | `(styleName: string, idMap: ModelIdMap) => Promise<void>` | Highlight specific IDs |
| `clear` | `(styleName?: string) => Promise<void>` | Clear highlights |
| `selection` | `Record<string, ModelIdMap>` | Current selections by style |
| `multiple` | `"none" \| "shiftKey" \| "ctrlKey"` | Multi-select modifier key |
| `zoomToSelection` | `boolean` | Auto-zoom to selected element |
| `zoomFactor` | `number` | Zoom distance multiplier |
| `autoToggle` | `Set<string>` | Styles that deselect on re-click |
| `mouseMoveThreshold` | `number` | Pixels before registering as drag (default: 5) |

#### ModelIdMap

```typescript
type ModelIdMap = Record<string, Set<number>>;
// Key: model.modelId
// Value: Set of EXPRESS IDs (local IDs within the model)
```

### OBC.Hider

Element visibility management.

| Method | Signature | Purpose |
|--------|-----------|---------|
| `set` | `(visible: boolean, idMap?: ModelIdMap) => Promise<void>` | Show/hide elements |
| `isolate` | `(idMap: ModelIdMap) => Promise<void>` | Show ONLY specified elements |

### OBCF.Clipper

Section planes through models.

| Method | Signature | Purpose |
|--------|-----------|---------|
| `create` | `(world: World) => ClippingPlane` | Create a new clipping plane |
| `delete` | `(plane: ClippingPlane) => void` | Remove a clipping plane |
| `deleteAll` | `() => void` | Remove all clipping planes |
| `enabled` | `boolean` | Enable/disable all clipping |

---

## web-ifc IfcAPI Methods

### Initialization

| Method | Signature | Purpose |
|--------|-----------|---------|
| `SetWasmPath` | `(path: string) => void` | Set WASM directory — MUST call BEFORE Init() |
| `Init` | `() => Promise<void>` | Load WASM module — MUST complete before any operation |
| `SetLogLevel` | `(level: LogLevel) => void` | Set verbosity (OFF, ERROR, INFO) |

### Model Lifecycle

| Method | Signature | Purpose |
|--------|-----------|---------|
| `OpenModel` | `(data: Uint8Array, settings?: LoaderSettings) => number` | Load IFC, returns modelID |
| `OpenModelFromCallback` | `(callback: Function) => number` | Stream-load large files |
| `CreateModel` | `(model: NewIfcModel, settings?: LoaderSettings) => number` | Create empty model |
| `CloseModel` | `(modelID: number) => void` | Free ALL WASM memory for model |
| `IsModelOpen` | `(modelID: number) => boolean` | Check if model loaded |
| `GetModelSchema` | `(modelID: number) => string` | Get IFC schema (IFC2X3/IFC4/IFC4X3) |

### LoaderSettings

```typescript
interface LoaderSettings {
  COORDINATE_TO_ORIGIN?: boolean;  // Center at origin (default: true)
  CIRCLE_SEGMENTS?: number;        // Tessellation detail for circles
  MEMORY_LIMIT?: number;           // Max WASM memory in bytes
  TAPE_SIZE?: number;              // Internal buffer size (default: 64MB)
}
```

### Entity Access

| Method | Signature | Purpose |
|--------|-----------|---------|
| `GetLine` | `(modelID, expressID, flatten?, inverse?) => object` | Get entity by ID |
| `GetAllLines` | `(modelID) => Vector<number>` | Get all entity IDs |
| `GetLineIDsWithType` | `(modelID, typeID) => Vector<number>` | Get entities by IFC type |

### Geometry

| Method | Signature | Purpose |
|--------|-----------|---------|
| `GetGeometry` | `(modelID, expressID) => IfcGeometry` | Geometry for one element |
| `GetFlatMesh` | `(modelID, expressID) => FlatMesh` | Mesh with all geometries |
| `GetVertexArray` | `(ptr, size) => Float32Array` | Extract vertices (6 floats each) |
| `GetIndexArray` | `(ptr, size) => Uint32Array` | Extract triangle indices |
| `StreamAllMeshes` | `(modelID, callback) => void` | Stream geometry incrementally |
| `StreamMeshes` | `(modelID, expressIDs[], callback) => void` | Stream specific elements |
| `LoadAllGeometry` | `(modelID) => Vector<FlatMesh>` | Load ALL at once (avoid for large models) |

### Properties Helper

```typescript
const props = ifcApi.properties;

await props.getItemProperties(modelID, expressID);
await props.getPropertySets(modelID, expressID, flatten?);
await props.getMaterialsProperties(modelID, expressID);
await props.getSpatialStructure(modelID);
```

### Vertex Data Format

Each vertex is 6 floats: `[x, y, z, nx, ny, nz]` (position + normal).

```
vertices.length / 6 = number of vertices
indices.length / 3 = number of triangles
```

---

## Three.js Key Classes for BIM Viewers

| Class | Purpose |
|-------|---------|
| `THREE.Scene` | Root container for all 3D objects |
| `THREE.PerspectiveCamera` | Standard perspective projection |
| `THREE.WebGLRenderer` | WebGL rendering — set `localClippingEnabled = true` for section planes |
| `THREE.BufferGeometry` | Efficient geometry from typed arrays |
| `THREE.BufferAttribute` | Wraps Float32Array/Uint32Array for GPU upload |
| `THREE.MeshPhongMaterial` | Material with diffuse + specular lighting |
| `THREE.Raycaster` | Mouse picking / intersection testing |
| `THREE.Plane` | Clipping plane definition |
| `THREE.Matrix4` | 4x4 transform matrix — use `.fromArray()` for IFC placement |
| `OrbitControls` | Camera navigation (orbit, pan, zoom) |
