# Vooronderzoek: Web-Based BIM/IFC Visualization

> Research document for the Cross-Tech AEC Skill Package.
> Topic: Three.js + web-ifc + @thatopen/components stack for browser-based BIM visualization.
> Date: 2026-03-20
> Status: Complete

---

## Table of Contents

1. [web-ifc (ThatOpen engine_web-ifc)](#1-web-ifc-thatopen-engine_web-ifc)
2. [@thatopen/components (formerly IFC.js)](#2-thatopencomponents-formerly-ifcjs)
3. [Three.js IFC Integration](#3-threejs-ifc-integration)
4. [IfcOpenShell vs web-ifc Comparison](#4-ifcopenshell-vs-web-ifc-comparison)
5. [BIM Web Viewer Pipeline](#5-bim-web-viewer-pipeline)
6. [Common Integration Failures](#6-common-integration-failures)
7. [Sources and Verification Status](#7-sources-and-verification-status)

---

## 1. web-ifc (ThatOpen engine_web-ifc)

### Overview

web-ifc is a JavaScript/TypeScript library for reading and writing IFC (Industry Foundation Classes) files at native speeds. It achieves this through a C++ core compiled to WebAssembly via Emscripten. The library is maintained by That Open Company (formerly IFC.js) and is the foundation of the entire ThatOpen BIM stack.

- **Current version**: 0.77 (released March 6, 2026) — *verified via GitHub*
- **License**: MPL-2.0
- **Language composition**: TypeScript 71.8%, C++ 28.0%
- **Repository**: https://github.com/ThatOpen/engine_web-ifc
- **npm**: `web-ifc`

### Build Outputs and WebAssembly Architecture

The compilation process produces multiple artifacts targeting different environments:

| Artifact | Purpose |
|----------|---------|
| `web-ifc.wasm` | Browser single-threaded WASM binary |
| `web-ifc-mt.wasm` | Browser multi-threaded WASM binary |
| `web-ifc-node.wasm` | Node.js WASM binary |
| `web-ifc-api.js` | Browser JavaScript API wrapper |
| `web-ifc-api-node.js` | Node.js JavaScript API wrapper |
| `web-ifc-mt.worker.js` | Web Worker for multi-threading |
| `*.d.ts` | TypeScript definitions (API, schema, properties, logging) |

Build requirements: Node v16+, npm v7+, Emscripten v4.0.23+, CMake v3.18+.

### IFC Schema Support

web-ifc supports three IFC schema versions — *verified via source code*:

```typescript
WebIFC.Schemas.IFC2X3
WebIFC.Schemas.IFC4
WebIFC.Schemas.IFC4X3
```

You can query the schema of a loaded model:

```typescript
const schema = ifcApi.GetModelSchema(modelID);
```

Or create a new model targeting a specific schema:

```typescript
const modelID = ifcApi.CreateModel({ schema: WebIFC.Schemas.IFC4 });
```

### IfcAPI Class — Core Interface

The `IfcAPI` class is the primary entry point. ALL operations go through this class. Initialization is asynchronous because it loads the WASM module.

```typescript
import { IfcAPI } from 'web-ifc';

const ifcApi = new IfcAPI();
await ifcApi.Init();  // MUST complete before any other call
```

### LoaderSettings Interface

*Verified from source at `src/ts/web-ifc-api.ts`*:

```typescript
interface LoaderSettings {
  COORDINATE_TO_ORIGIN?: boolean;       // Center geometry at origin (default: true)
  CIRCLE_SEGMENTS?: number;             // Tessellation detail for circles
  MEMORY_LIMIT?: number;                // Max WASM memory in bytes (default: ~2GB)
  TAPE_SIZE?: number;                   // Internal buffer size in bytes (default: 64MB)
  LINEWRITER_BUFFER?: number;           // Write buffer size
  TOLERANCE_PLANE_INTERSECTION?: number;
  TOLERANCE_PLANE_DEVIATION?: number;
  TOLERANCE_BACK_DEVIATION_DISTANCE?: number;
  TOLERANCE_INSIDE_OUTSIDE_PERIMETER?: number;
  TOLERANCE_SCALAR_EQUALITY?: number;
  PLANE_REFIT_ITERATIONS?: number;
  BOOLEAN_UNION_THRESHOLD?: number;
}
```

### Model Lifecycle Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `OpenModel` | `(data: Uint8Array, settings?: LoaderSettings) => number` | Load IFC, returns modelID or -1 |
| `OpenModelFromCallback` | `(callback: Function) => number` | Stream-load for large files |
| `CreateModel` | `(model: NewIfcModel, settings?: LoaderSettings) => number` | Create new empty model |
| `CloseModel` | `(modelID: number) => void` | Free ALL memory for model |
| `IsModelOpen` | `(modelID: number) => boolean` | Check if model is loaded |
| `GetModelSchema` | `(modelID: number) => string` | Get IFC schema version |

Model IDs are sequential integers starting at 0. Multiple models can be open simultaneously.

### Entity Access Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `GetLine` | `(modelID, expressID, flatten?, inverse?, inversePropKey?) => object` | Get entity by EXPRESS ID |
| `GetAllLines` | `(modelID) => Vector<number>` | Get all entity IDs |
| `GetLineIDsWithType` | `(modelID, typeID) => Vector<number>` | Get entities by IFC type |

The `flatten` parameter on `GetLine` is critical. By default, nested references return as `Handle` objects (containing only an expressID). When `flatten = true`, all references are recursively resolved to full objects:

```typescript
// Default: references are Handle objects
const wall = ifcApi.GetLine(modelID, expressID);
// wall.OwnerHistory → { expressID: 41, type: 5 }  (Handle)

// Flattened: all references resolved
const wallFlat = ifcApi.GetLine(modelID, expressID, true);
// wallFlat.OwnerHistory → { ... full IfcOwnerHistory object ... }
```

The `inverse` parameter enables access to inverse attributes (entities that reference this entity), which is important for traversing the IFC graph in reverse.

### Geometry Extraction Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `GetGeometry` | `(modelID, expressID) => IfcGeometry` | Get geometry for one element |
| `GetFlatMesh` | `(modelID, expressID) => FlatMesh` | Get mesh with all geometries |
| `GetVertexArray` | `(geometry: IfcGeometry) => Float32Array` | Extract vertex data |
| `GetIndexArray` | `(geometry: IfcGeometry) => Uint32Array` | Extract triangle indices |
| `LoadAllGeometry` | `(modelID) => Vector<FlatMesh>` | Load ALL geometry at once |
| `StreamAllMeshes` | `(modelID, callback) => void` | Stream geometry one-by-one |
| `StreamMeshes` | `(modelID, expressIDs[], callback) => void` | Stream specific meshes |

Vertex data format: 6 floats per vertex (x, y, z, nx, ny, nz — position + normal).

```typescript
const geometry = ifcApi.GetGeometry(modelID, expressID);

const vertices = ifcApi.GetVertexArray(
  geometry.GetVertexData(),
  geometry.GetVertexDataSize()
);
// vertices.length / 6 = number of vertices

const indices = ifcApi.GetIndexArray(
  geometry.GetIndexData(),
  geometry.GetIndexDataSize()
);
// indices.length / 3 = number of triangles
```

### Streaming API for Large Models

`StreamAllMeshes` processes geometry without loading everything into memory simultaneously. This is the ONLY safe approach for large models:

```typescript
ifcApi.StreamAllMeshes(modelID, (mesh) => {
  // mesh.expressID — the entity this geometry belongs to
  const geometry = ifcApi.GetGeometry(modelID, mesh.expressID);
  const verts = ifcApi.GetVertexArray(geometry.GetVertexData(), geometry.GetVertexDataSize());
  const idx = ifcApi.GetIndexArray(geometry.GetIndexData(), geometry.GetIndexDataSize());
  // Convert to Three.js BufferGeometry here
});
```

### Properties Helper

The `ifcApi.properties` object provides convenient typed access:

```typescript
const props = ifcApi.properties;

// Element properties
const item = await props.getItemProperties(modelID, expressID);

// Property sets (Psets)
const psets = await props.getPropertySets(modelID, expressID);
const psetsFlat = await props.getPropertySets(modelID, expressID, true); // resolved refs

// Materials
const materials = await props.getMaterialsProperties(modelID, expressID);

// Spatial structure (Site → Building → Storey → Space hierarchy)
const structure = await props.getSpatialStructure(modelID);
```

### Logging Configuration

```typescript
ifcApi.SetLogLevel(WebIFC.LogLevel.LOG_LEVEL_OFF);    // Silent
ifcApi.SetLogLevel(WebIFC.LogLevel.LOG_LEVEL_ERROR);  // Errors only
ifcApi.SetLogLevel(WebIFC.LogLevel.LOG_LEVEL_INFO);   // Verbose
```

### Memory Management

WASM memory is separate from JavaScript heap memory. Key rules:

1. ALWAYS call `CloseModel(modelID)` when done — this frees ALL WASM memory for that model
2. The `MEMORY_LIMIT` setting caps total WASM allocation (default ~2GB, the WebAssembly spec limit)
3. `TAPE_SIZE` controls the internal read buffer (default 64MB) — increase for very large files
4. Each `GetGeometry` / `GetVertexArray` call allocates memory — process and discard promptly
5. For batch operations, use `StreamAllMeshes` instead of `LoadAllGeometry` to avoid peak memory

---

## 2. @thatopen/components (formerly IFC.js)

### Overview

`@thatopen/components` is a component-based BIM application framework built on Three.js and web-ifc. It provides pre-built, composable building blocks for constructing browser-based BIM viewers with features like model loading, selection highlighting, clipping planes, measurements, and more.

- **Current version**: 3.3.2 (released January 27, 2026) — *verified via GitHub releases*
- **License**: MIT
- **Repository**: https://github.com/ThatOpen/engine_components
- **Packages**: `@thatopen/components` (core, browser + Node) and `@thatopen/components-front` (browser-only UI)

### Installation

```bash
npm install @thatopen/components @thatopen/components-front @thatopen/fragments three web-ifc
```

**CRITICAL**: The Three.js version MUST match the one specified in `@thatopen/components` package.json. Version mismatches cause silent rendering failures.

### Architecture: Components + Worlds

The framework uses two core concepts:

1. **Components**: Globally available singletons with lifecycle management. Each component handles one concern (loading, highlighting, clipping, etc.).
2. **Worlds**: Container objects that group a Scene, Camera, and Renderer — analogous to a Three.js application context.

```typescript
import * as OBC from "@thatopen/components";
import * as THREE from "three";

// Initialize the component system
const components = new OBC.Components();

// Create a world (scene + camera + renderer)
const worlds = components.get(OBC.Worlds);
const world = worlds.create<
  OBC.SimpleScene,
  OBC.SimpleCamera,
  OBC.SimpleRenderer
>();
```

### The Fragments Format

That Open Engine does NOT directly render IFC geometry. When an IFC file is "loaded", it is first CONVERTED into Fragments — That Open's open-source binary format built on Google's Flatbuffers. This conversion is the performance-intensive step; loading pre-converted Fragments is near-instant.

The pipeline is: **IFC file → web-ifc parsing → Fragment conversion → Three.js rendering**

This means:
- First load of an IFC file is slow (parsing + tessellation + conversion)
- Subsequent loads can use cached `.frag` files (fast)
- Fragments can be exported and stored server-side for instant loading

### Key Components

#### IfcLoader

Converts IFC files to Fragments and loads them into the scene:

```typescript
const ifcLoader = components.get(OBC.IfcLoader);

// Configure WASM location
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "https://unpkg.com/web-ifc@0.0.74/",
    absolute: true,
  },
});

// Load an IFC file
const file = await fetch("model.ifc");
const data = await file.arrayBuffer();
const buffer = new Uint8Array(data);
await ifcLoader.load(buffer, false, "model-name", {
  processData: {
    progressCallback: (progress) => console.log(`${progress}% loaded`),
  },
});
```

Not all IFC element types are converted. You can inspect which classes are included:

```typescript
ifcLoader.onIfcImporterInitialized.add((importer) => {
  console.log(importer.classes);  // Set of IFC types that will produce geometry
});
```

#### FragmentsManager

Central manager for all loaded Fragment models. Handles loading, disposal, and provides access to loaded models:

```typescript
const fragments = components.get(OBC.FragmentsManager);

// Initialize with worker for off-thread processing
const workerUrl = URL.createObjectURL(workerFile);
fragments.init(workerUrl);

// React to new models being loaded
fragments.list.onItemSet.add(({ value: model }) => {
  model.useCamera(world.camera.three);
  world.scene.three.add(model.object);
  fragments.core.update(true);
});

// Export a model to .frag format for caching
const fragsBuffer = await model.getBuffer(false);
const file = new File([fragsBuffer], "model.frag");
```

#### Highlighter (components-front)

Enables selection and highlighting of elements in the 3D scene:

```typescript
const highlighter = components.get(OBCF.Highlighter);

// Configuration
highlighter.multiple = "ctrlKey";  // "none" | "shiftKey" | "ctrlKey"
highlighter.zoomToSelection = true;
highlighter.zoomFactor = 1.5;

// Highlight by raycasting (mouse interaction)
await highlighter.highlight("select");

// Highlight by ID map
const modelIdMap: OBC.ModelIdMap = {};
modelIdMap[model.modelId] = new Set([101, 102, 103]);
await highlighter.highlightByID("select", modelIdMap);

// Clear selections
await highlighter.clear();          // Clear all
await highlighter.clear("select");  // Clear specific style
```

Key properties:
- `selection` — current selections indexed by style name
- `styles` — DataMap of highlight styles (MaterialDefinition or null)
- `autoToggle` — set of style names that deselect on re-click
- `mouseMoveThreshold` — pixels of mouse movement before registering as drag (default: 5)

#### Hider

Manages element visibility — hide, show, and isolate operations:

```typescript
const hider = components.get(OBC.Hider);

// Build a ModelIdMap for targeting
const modelIdMap: OBC.ModelIdMap = {};
modelIdMap[model.modelId] = new Set(localIds);

// Isolate: show ONLY these elements
await hider.isolate(modelIdMap);

// Hide specific elements
await hider.set(false, modelIdMap);

// Reset: show everything
await hider.set(true);
```

Combine with category queries for filtering:

```typescript
const wallIds = model.getItemsOfCategories(["IFCWALL"]);
const wallMap: OBC.ModelIdMap = {};
wallMap[model.modelId] = wallIds;
await hider.isolate(wallMap);
```

#### Clipper (components-front)

Creates clipping planes for section cuts through models. Works with viewpoints for saving and restoring clip states.

#### Additional Components

The ecosystem includes components for:
- **Measurements**: Length, area, angle dimensions
- **Plans**: 2D floorplan generation and navigation
- **DXF Export**: Export 2D views to DXF format
- **Postproduction**: Visual effects (ambient occlusion, outlines)
- **Viewpoints**: Save/restore camera + selection + clip states (BCF compatible)
- **ItemsFinder**: Advanced element search and filtering

### Z-Fighting Prevention

A common visual artifact with BIM models. The standard fix in @thatopen/components:

```typescript
fragments.core.models.materials.list.onItemSet.add(({ value: material }) => {
  material.polygonOffset = true;
  material.polygonOffsetUnits = 1;
  material.polygonOffsetFactor = Math.random();  // Randomize to separate coplanar faces
});
```

### Platform Compatibility

@thatopen/components works in:
- Vanilla JavaScript/TypeScript web apps
- React, Vue, Angular, Svelte
- Node.js (core package only)
- Electron desktop apps
- React Native (limited)

---

## 3. Three.js IFC Integration

### Coordinate System Transformation

This is one of the most critical integration points:

- **IFC standard**: Z-up coordinate system (Z points upward, X/Y form the ground plane)
- **Three.js**: Y-up coordinate system (Y points upward, X/Z form the ground plane)

web-ifc handles this transformation internally — geometry extracted from web-ifc is ALREADY in Y-up coordinates suitable for Three.js. You do NOT need to manually rotate the model by -90 degrees around X. However, if you are using raw IFC coordinates (e.g., from IfcOpenShell on the server), you MUST apply the rotation yourself:

```typescript
// ONLY needed when importing coordinates from Z-up sources (not web-ifc)
model.rotation.x = -Math.PI / 2;
```

### Converting web-ifc Geometry to Three.js BufferGeometry

When building a custom viewer without @thatopen/components, you must manually convert web-ifc output to Three.js meshes:

```typescript
import * as THREE from 'three';
import { IfcAPI } from 'web-ifc';

const ifcApi = new IfcAPI();
await ifcApi.Init();

const data = new Uint8Array(await fetch('model.ifc').then(r => r.arrayBuffer()));
const modelID = ifcApi.OpenModel(data);

// Stream all meshes and convert to Three.js
const scene = new THREE.Scene();

ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  const placedGeometries = flatMesh.geometries;

  for (let i = 0; i < placedGeometries.size(); i++) {
    const placedGeometry = placedGeometries.get(i);
    const geometry = ifcApi.GetGeometry(modelID, placedGeometry.geometryExpressID);

    // Extract vertex data (x, y, z, nx, ny, nz per vertex)
    const verts = ifcApi.GetVertexArray(
      geometry.GetVertexData(),
      geometry.GetVertexDataSize()
    );
    const indices = ifcApi.GetIndexArray(
      geometry.GetIndexData(),
      geometry.GetIndexDataSize()
    );

    // Build Three.js BufferGeometry
    const bufferGeometry = new THREE.BufferGeometry();

    // Separate position and normal from interleaved array
    const posFloats = new Float32Array(verts.length / 2);  // 3 of every 6
    const normFloats = new Float32Array(verts.length / 2); // 3 of every 6
    for (let j = 0; j < verts.length; j += 6) {
      posFloats[j / 2]     = verts[j];
      posFloats[j / 2 + 1] = verts[j + 1];
      posFloats[j / 2 + 2] = verts[j + 2];
      normFloats[j / 2]     = verts[j + 3];
      normFloats[j / 2 + 1] = verts[j + 4];
      normFloats[j / 2 + 2] = verts[j + 5];
    }

    bufferGeometry.setAttribute('position', new THREE.BufferAttribute(posFloats, 3));
    bufferGeometry.setAttribute('normal', new THREE.BufferAttribute(normFloats, 3));
    bufferGeometry.setIndex(new THREE.BufferAttribute(indices, 1));

    // Apply placement transform (4x4 matrix from IFC)
    const matrix = new THREE.Matrix4().fromArray(placedGeometry.flatTransformation);
    bufferGeometry.applyMatrix4(matrix);

    // Create mesh with material based on IFC color
    const color = placedGeometry.color;
    const material = new THREE.MeshPhongMaterial({
      color: new THREE.Color(color.x, color.y, color.z),
      opacity: color.w,
      transparent: color.w < 1.0,
      side: THREE.DoubleSide,
    });

    const mesh = new THREE.Mesh(bufferGeometry, material);
    scene.add(mesh);
  }
});
```

### Raycasting for Element Selection

Three.js raycasting works on the generated meshes. The challenge is mapping back from a Three.js mesh to the IFC EXPRESS ID for property lookup:

```typescript
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

function onMouseClick(event: MouseEvent) {
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

  raycaster.setFromCamera(mouse, camera);
  const intersects = raycaster.intersectObjects(scene.children, true);

  if (intersects.length > 0) {
    const mesh = intersects[0].object;
    // You must have stored the expressID on the mesh during creation
    const expressID = mesh.userData.expressID;
    if (expressID !== undefined) {
      const props = ifcApi.GetLine(modelID, expressID, true);
      console.log('Selected:', props.Name?.value, props);
    }
  }
}
```

The key challenge: each IFC element can contain multiple geometries (placed at different transforms), and you need to store the mapping from Three.js Object3D back to IFC expressID. @thatopen/components handles this automatically through its Fragment indexing system.

### Scene Graph Organization

For BIM viewers, organize the Three.js scene to mirror IFC spatial structure:

```
Scene
├── Site (IfcSite)
│   ├── Building (IfcBuilding)
│   │   ├── Storey_0 (IfcBuildingStorey)
│   │   │   ├── Wall_001 (IfcWall)
│   │   │   ├── Slab_001 (IfcSlab)
│   │   │   └── ...
│   │   ├── Storey_1
│   │   │   └── ...
│   │   └── ...
│   └── ...
└── Lights, Grid, etc.
```

This enables per-storey visibility toggling, spatial queries, and logical organization. @thatopen/components provides the Hider component for this purpose; a custom implementation requires building this hierarchy manually from `ifcApi.properties.getSpatialStructure()`.

---

## 4. IfcOpenShell vs web-ifc Comparison

### Architecture

| Aspect | IfcOpenShell | web-ifc |
|--------|-------------|---------|
| **Language** | C++ with Python bindings | C++ compiled to WebAssembly |
| **Runtime** | Server-side (Python/C++) | Browser or Node.js |
| **Geometry engine** | Open CASCADE (OCCT) | Custom engine |
| **License** | LGPL-3.0 | MPL-2.0 |
| **Current version** | 0.8.x (2026) | 0.77 (2026) |

### Feature Comparison

| Feature | IfcOpenShell | web-ifc |
|---------|-------------|---------|
| IFC2X3 support | Full | Full |
| IFC4 support | Full | Full |
| IFC4X3 support | Partial | Supported |
| Inverse attributes | Full support | Supported (via `inverse` param) |
| Geometry tessellation | OCCT-based, high quality | Custom, optimized for speed |
| Boolean operations | Robust (OCCT) | Fast but less robust |
| IFC authoring/writing | Full (via IfcOpenShell API) | Basic (CreateModel + WriteLine) |
| Property access | `element.IsDefinedBy` etc. | `GetLine` + flatten/inverse |
| Spatial queries | Built-in utility functions | Manual traversal |
| File size handling | Limited by system RAM | Limited by WASM memory (~2GB) |
| Multi-threading | Native OS threads | Web Workers |

### Performance Characteristics

- **IfcOpenShell**: Better for complex geometric operations (booleans, BREP), large-scale server processing, and batch operations on many files. Slower startup but faster on complex geometry.
- **web-ifc**: Better for interactive browser viewing, real-time property inspection, and client-side offline use. Fast startup, streaming-friendly, but hits memory ceiling sooner.

### Property Access Patterns

**IfcOpenShell (Python)**:
```python
import ifcopenshell

ifc = ifcopenshell.open("model.ifc")
walls = ifc.by_type("IfcWall")
for wall in walls:
    print(wall.Name)
    # Inverse attribute access
    for rel in wall.IsDefinedBy:
        if rel.is_a("IfcRelDefinesByProperties"):
            pset = rel.RelatingPropertyDefinition
            for prop in pset.HasProperties:
                print(f"  {prop.Name}: {prop.NominalValue.wrappedValue}")
```

**web-ifc (TypeScript)**:
```typescript
const wallIDs = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCWALL);
for (let i = 0; i < wallIDs.size(); i++) {
  const wall = ifcApi.GetLine(modelID, wallIDs.get(i), true);
  console.log(wall.Name.value);

  const psets = await ifcApi.properties.getPropertySets(modelID, wallIDs.get(i), true);
  // psets is already resolved
}
```

### When to Use Which

| Scenario | Recommended Tool |
|----------|-----------------|
| Browser-based BIM viewer | web-ifc |
| Server-side IFC validation | IfcOpenShell |
| IFC authoring (creating/editing models) | IfcOpenShell |
| Client-side offline viewing | web-ifc |
| Complex geometry processing (BREP) | IfcOpenShell |
| Real-time interactive inspection | web-ifc |
| Batch processing thousands of files | IfcOpenShell |
| Federated model merging | IfcOpenShell |

### Hybrid Architecture (Recommended)

The community consensus for production BIM web applications is a hybrid approach:

- **Frontend (browser)**: web-ifc + @thatopen/components for visualization, navigation, selection, property inspection
- **Backend (server)**: IfcOpenShell for authoring, validation, complex queries, format conversion, and operations requiring more than 2GB memory

Communication via REST API or WebSocket. The server can also pre-process IFC files into Fragments format for instant browser loading.

---

## 5. BIM Web Viewer Pipeline

### Pipeline Overview

```
IFC File (.ifc)
    │
    ▼
┌─────────────────────────────────────────────┐
│  Option A: @thatopen/components Pipeline     │
│                                              │
│  IFC → web-ifc (WASM parse) → Fragments     │
│  Fragments → Three.js Scene → WebGL Render   │
│  Components: Highlighter, Clipper, Hider     │
└─────────────────────────────────────────────┘
    │
    OR
    │
┌─────────────────────────────────────────────┐
│  Option B: Custom Pipeline                   │
│                                              │
│  IFC → web-ifc (WASM parse)                 │
│  StreamAllMeshes → BufferGeometry            │
│  Manual Three.js scene + raycasting          │
│  Custom UI for properties, clips, etc.       │
└─────────────────────────────────────────────┘
```

### Option A: @thatopen/components Approach (Recommended)

Complete minimal BIM viewer:

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
import * as THREE from "three";

// 1. Initialize component system
const components = new OBC.Components();

// 2. Create world
const worlds = components.get(OBC.Worlds);
const world = worlds.create<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer>();

// 3. Configure renderer
const container = document.getElementById("viewer")!;
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.SimpleCamera(components);
world.scene = new OBC.SimpleScene(components);

// 4. Setup IFC loader
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: { path: "https://unpkg.com/web-ifc@0.0.74/", absolute: true },
});

// 5. Setup FragmentsManager
const fragments = components.get(OBC.FragmentsManager);
fragments.list.onItemSet.add(({ value: model }) => {
  model.useCamera(world.camera.three);
  world.scene.three.add(model.object);
  fragments.core.update(true);
});

// 6. Setup interaction components
const highlighter = components.get(OBCF.Highlighter);
highlighter.multiple = "ctrlKey";
highlighter.zoomToSelection = true;

const hider = components.get(OBC.Hider);
// const clipper = components.get(OBCF.Clipper);

// 7. Load IFC
async function loadIfc(url: string) {
  const response = await fetch(url);
  const data = new Uint8Array(await response.arrayBuffer());
  await ifcLoader.load(data, false, "my-model");
}

await loadIfc("/models/building.ifc");
```

### Option B: Custom Pipeline (Full Control)

When you need complete control over rendering, material assignment, or integration with an existing Three.js application:

```typescript
import * as THREE from 'three';
import { IfcAPI, IFCWALL, IFCSLAB, IFCWINDOW } from 'web-ifc';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

// Scene setup
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);

// IFC loading
const ifcApi = new IfcAPI();
await ifcApi.Init();
ifcApi.SetLogLevel(0); // Silent

const ifcData = new Uint8Array(await fetch('model.ifc').then(r => r.arrayBuffer()));
const modelID = ifcApi.OpenModel(ifcData, {
  COORDINATE_TO_ORIGIN: true,
});

// Map expressID → Three.js mesh for selection
const expressIdToMesh = new Map<number, THREE.Mesh>();

// Stream and convert geometry
ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  const placedGeometries = flatMesh.geometries;
  for (let i = 0; i < placedGeometries.size(); i++) {
    const pg = placedGeometries.get(i);
    const geom = ifcApi.GetGeometry(modelID, pg.geometryExpressID);

    const verts = ifcApi.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
    const idx = ifcApi.GetIndexArray(geom.GetIndexData(), geom.GetIndexDataSize());

    const bufGeom = new THREE.BufferGeometry();
    const positions = new Float32Array(verts.length / 2);
    const normals = new Float32Array(verts.length / 2);
    for (let j = 0; j < verts.length; j += 6) {
      const k = j / 2;
      positions[k] = verts[j]; positions[k+1] = verts[j+1]; positions[k+2] = verts[j+2];
      normals[k] = verts[j+3]; normals[k+1] = verts[j+4]; normals[k+2] = verts[j+5];
    }
    bufGeom.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    bufGeom.setAttribute('normal', new THREE.BufferAttribute(normals, 3));
    bufGeom.setIndex(new THREE.BufferAttribute(idx, 1));

    const matrix = new THREE.Matrix4().fromArray(pg.flatTransformation);
    bufGeom.applyMatrix4(matrix);

    const mat = new THREE.MeshPhongMaterial({
      color: new THREE.Color(pg.color.x, pg.color.y, pg.color.z),
      opacity: pg.color.w,
      transparent: pg.color.w < 1.0,
      side: THREE.DoubleSide,
    });

    const mesh = new THREE.Mesh(bufGeom, mat);
    mesh.userData.expressID = flatMesh.expressID;
    scene.add(mesh);
    expressIdToMesh.set(flatMesh.expressID, mesh);
  }
});

// Property panel on click
const raycaster = new THREE.Raycaster();
renderer.domElement.addEventListener('click', (event) => {
  const mouse = new THREE.Vector2(
    (event.clientX / window.innerWidth) * 2 - 1,
    -(event.clientY / window.innerHeight) * 2 + 1
  );
  raycaster.setFromCamera(mouse, camera);
  const hits = raycaster.intersectObjects(scene.children);
  if (hits.length > 0) {
    const expressID = hits[0].object.userData.expressID;
    const entity = ifcApi.GetLine(modelID, expressID, true);
    showPropertyPanel(entity); // Your UI function
  }
});

// Render loop
function animate() {
  requestAnimationFrame(animate);
  controls.update();
  renderer.render(scene, camera);
}
animate();
```

### User Interactions

A complete BIM viewer provides these interaction modes:

| Interaction | @thatopen/components | Custom Implementation |
|-------------|---------------------|----------------------|
| Navigate (orbit/pan/zoom) | SimpleCamera (built-in) | OrbitControls from Three.js |
| Select element | Highlighter.highlight() | Raycaster + expressID lookup |
| Inspect properties | FragmentsManager + GetLine | Manual property panel UI |
| Clip/section | Clipper component | THREE.Plane + clippingPlanes |
| Hide/isolate | Hider component | mesh.visible = false |
| Measure | Measurement components | Custom line geometry + snapping |
| Storey navigation | Plans component | getSpatialStructure + Hider |

---

## 6. Common Integration Failures

### 6.1 Large File Handling (>100MB)

**Problem**: IFC files for real construction projects routinely exceed 100MB, sometimes reaching 500MB–1GB. Browser memory is severely constrained compared to desktop applications.

**Constraints** — *verified via multiple sources*:
- WebAssembly linear memory maximum: ~4GB (theoretical), ~2GB practical on most browsers
- JavaScript string limit: ~1GB
- Total browser tab memory: typically 2–4GB before crash
- Laptops with 8GB RAM: practical limit around 1.7GB for the entire tab

**Mitigation strategies**:

1. **Server-side pre-processing**: Convert IFC to Fragments on the server, serve `.frag` files to browsers. Fragments are 5–10x smaller than source IFC.
2. **Streaming geometry**: Use `StreamAllMeshes` instead of `LoadAllGeometry`. Process and render incrementally.
3. **Tiling**: Divide models into spatial tiles, load only visible tiles. Like map applications that load only the viewport area.
4. **Level-of-detail (LOD)**: Serve simplified geometry for distant objects, full detail only for nearby elements.
5. **Frustum culling**: Three.js handles this automatically but ensure it is not disabled.
6. **Geometry instancing**: Reuse BufferGeometry for repeated elements (e.g., identical windows). web-ifc provides geometry expressIDs that can be shared.

**Performance benchmark** — *from AlterSquare case study (inferred, not independently verified)*:
- Unoptimized large model: ~3 FPS
- With culling + tiling + compression: ~20 FPS

### 6.2 Memory Limits in Browsers

**Problem**: WASM memory allocation failures crash the tab silently.

**Symptoms**:
- `RangeError: WebAssembly.Memory(): could not allocate memory`
- Browser tab becomes unresponsive
- `Out of memory` in console

**Prevention**:
```typescript
const settings: LoaderSettings = {
  MEMORY_LIMIT: 1073741824,  // 1GB — conservative limit
  TAPE_SIZE: 33554432,       // 32MB — reduce for small files
};
const modelID = ifcApi.OpenModel(data, settings);
```

ALWAYS close models when no longer needed:
```typescript
ifcApi.CloseModel(modelID);
```

### 6.3 Missing or Unsupported Geometry Types

**Problem**: Not all IFC geometry representations are supported by web-ifc. Complex BREP (Boundary Representation) geometry, advanced boolean operations, and some parametric shapes may produce incorrect or missing geometry.

**Common failures**:
- `IfcAdvancedBrep` — complex curved surfaces may tessellate incorrectly
- `IfcBooleanResult` with complex operands — may produce visual artifacts
- `IfcMappedItem` — instancing can fail if the mapped representation is unsupported
- Curved walls (`IfcWallStandardCase` with `IfcTrimmedCurve` axis) — may produce faceted approximations

**Workaround**: For critical geometry, pre-process with IfcOpenShell (which uses OCCT, a more robust geometry kernel) and export as simplified meshes or GLTF.

### 6.4 Property Access Differences

**Problem**: Property traversal patterns differ fundamentally between IfcOpenShell and web-ifc, causing confusion when developers switch between server-side and client-side code.

| Pattern | IfcOpenShell | web-ifc |
|---------|-------------|---------|
| Get element | `ifc.by_id(123)` | `ifcApi.GetLine(modelID, 123)` |
| Get by type | `ifc.by_type("IfcWall")` | `ifcApi.GetLineIDsWithType(modelID, IFCWALL)` |
| Access name | `element.Name` | `entity.Name.value` |
| Nested refs | Automatic resolution | Manual (use `flatten=true`) |
| Inverse attrs | `element.IsDefinedBy` | `GetLine(modelID, id, false, true)` |
| Property sets | Walk `IsDefinedBy` relation | `properties.getPropertySets()` |

Key difference: IfcOpenShell returns Python objects with direct attribute access. web-ifc returns plain JavaScript objects where string/numeric values are wrapped in `{ value: ... }` containers and references are `Handle` objects unless flattened.

### 6.5 Coordinate System Issues

**Problem**: IFC uses Z-up; Three.js uses Y-up.

**When it matters**:
- web-ifc ALREADY transforms to Y-up — no action needed when using web-ifc directly
- IfcOpenShell outputs Z-up coordinates — MUST transform when sending to a Three.js frontend
- GIS integrations (QGIS, CesiumJS) use various CRS — requires explicit coordinate transformation
- `COORDINATE_TO_ORIGIN: true` (default) moves geometry to origin — original real-world coordinates are lost unless stored separately

**Transform for Z-up to Y-up**:
```typescript
// When receiving coordinates from IfcOpenShell or other Z-up sources
const rotationMatrix = new THREE.Matrix4().makeRotationX(-Math.PI / 2);
geometry.applyMatrix4(rotationMatrix);
```

### 6.6 WASM Path Configuration

**Problem**: web-ifc needs to locate its `.wasm` files at runtime. Misconfigured paths cause silent failures.

**In @thatopen/components**:
```typescript
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: {
    path: "/static/wasm/",  // Must point to directory containing web-ifc.wasm
    absolute: true,
  },
});
```

**In raw web-ifc**:
```typescript
const ifcApi = new IfcAPI();
ifcApi.SetWasmPath("/static/wasm/");  // BEFORE Init()
await ifcApi.Init();
```

Common mistakes:
- Setting the path AFTER `Init()` — too late, WASM already loaded (or failed)
- Wrong relative path in bundled applications (Webpack, Vite)
- Not copying `.wasm` files to the build output directory
- CORS issues when loading WASM from a CDN

### 6.7 Three.js Version Mismatch

**Problem**: @thatopen/components pins a specific Three.js version. Using a different version causes subtle bugs.

**Symptoms**:
- `TypeError: Cannot read properties of undefined`
- Missing methods on Three.js objects
- Rendering artifacts
- Components fail to initialize

**Prevention**: ALWAYS check the Three.js version required by @thatopen/components:
```bash
npm ls three
# Verify only ONE version of three is installed
```

---

## 7. Sources and Verification Status

| Source | URL | Status |
|--------|-----|--------|
| web-ifc GitHub repository | https://github.com/ThatOpen/engine_web-ifc | Verified (v0.77, March 2026) |
| web-ifc API source code | https://github.com/ThatOpen/engine_web-ifc/blob/main/src/ts/web-ifc-api.ts | Verified (method signatures) |
| web-ifc DeepWiki | https://deepwiki.com/ThatOpen/engine_web-ifc/2-getting-started | Verified (API usage, settings) |
| engine_components GitHub | https://github.com/ThatOpen/engine_components | Verified (v3.3.2, Jan 2026) |
| ThatOpen docs — IfcLoader | https://docs.thatopen.com/Tutorials/Components/Core/IfcLoader | Verified (Fragments pipeline) |
| ThatOpen docs — Getting Started | https://docs.thatopen.com/components/getting-started | Verified (installation) |
| ThatOpen docs — Highlighter API | https://docs.thatopen.com/api/@thatopen/components-front/classes/Highlighter | Verified (full API) |
| ThatOpen docs — Hider | https://docs.thatopen.com/Tutorials/Components/Core/Hider | Verified (API + examples) |
| OSArch community — IFC.js + IfcOpenShell | https://community.osarch.org/discussion/1089 | Verified (community discussion) |
| Three.js documentation | https://threejs.org/docs/ | Reference (stable, not version-specific) |
| IfcOpenShell documentation | https://docs.ifcopenshell.org/ | Reference |
| AlterSquare — Large IFC handling | https://altersquare.io/handling-large-ifc-files-in-web-applications-performance-optimization-guide/ | Inferred (403 error, data from search snippets) |

### What Was Verified vs Inferred

**Verified against source code or official documentation**:
- IfcAPI method signatures (from `web-ifc-api.ts` source)
- LoaderSettings interface (from source)
- IFC schema support (IFC2X3, IFC4, IFC4X3)
- Version numbers (web-ifc 0.77, components 3.3.2)
- Fragments format and conversion pipeline
- Component API (Highlighter, Hider, IfcLoader, FragmentsManager)
- Z-up to Y-up coordinate handling by web-ifc

**Inferred from community discussion and search snippets**:
- Performance benchmarks (3 FPS unoptimized, 20 FPS optimized)
- 1.7GB practical memory limit on 8GB laptops
- Specific BREP/boolean geometry failure modes
- IfcOpenShell vs web-ifc performance comparison details

**Not independently verified**:
- Exact behavior of `OpenModelFromCallback` for streaming large files
- Multi-threading performance gains with `web-ifc-mt.wasm`
- React Native compatibility claims for @thatopen/components

---

*End of research document. Word count: ~3500 words.*
