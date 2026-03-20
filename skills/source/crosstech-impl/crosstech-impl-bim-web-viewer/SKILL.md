---
name: crosstech-impl-bim-web-viewer
description: >
  Use when building an end-to-end BIM web viewer: from IFC file to interactive
  browser application with navigation, selection, property inspection, and clipping.
  Prevents the common mistake of building a custom viewer when @thatopen/components
  provides production-ready functionality out of the box.
  Covers the full pipeline: IFC parsing, Three.js rendering, UI interaction,
  section planes, annotations, and performance optimization for large models.
  Keywords: BIM viewer, web viewer, IFC viewer, @thatopen/components, Three.js,
  section plane, property panel, model navigation, web-ifc, BIM web application.
license: MIT
compatibility: "Designed for Claude Code. Covers @thatopen/components 3.x, Three.js r160+, web-ifc 0.0.77."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-impl-bim-web-viewer

## Technology Boundary

| Aspect | Side A: BIM/IFC Data | Side B: Web Browser |
|--------|---------------------|---------------------|
| Format | IFC STEP file (.ifc) — IFC4 / IFC4X3 | WebGL rendering via Three.js |
| Coordinates | Z-up (IFC standard) | Y-up (Three.js / WebGL) |
| Data model | EXPRESS schema, entity graph | JavaScript objects, typed arrays |
| Units | Millimeters or meters (per IfcProject) | Three.js scene units (unitless) |
| Size limit | No limit (files reach 500MB–1GB) | ~2GB WASM memory, ~4GB tab total |

**The Bridge**: @thatopen/components (v3.x) — converts IFC to Fragments format via web-ifc WASM, then renders through Three.js. Handles coordinate transformation, geometry tessellation, and entity-to-mesh mapping automatically.

**Directionality**: IFC → Web browser (one-way rendering + interactive inspection). Writing IFC from the browser is NOT covered by this skill.

---

## Decision Tree: Which Approach?

```
Need a BIM web viewer?
├── Using standard BIM features (navigate, select, clip, inspect)?
│   └── YES → Use @thatopen/components (Option A)
├── Integrating into existing Three.js app with custom rendering?
│   └── YES → Use custom web-ifc + Three.js pipeline (Option B)
├── Need server-side IFC processing (validation, authoring)?
│   └── YES → Use IfcOpenShell on backend + web-ifc on frontend (Hybrid)
└── File size > 200MB?
    └── YES → ALWAYS pre-convert to Fragments on server (see Large Model Handling)
```

---

## Option A: @thatopen/components Viewer (Recommended)

### Installation

```bash
npm install @thatopen/components @thatopen/components-front @thatopen/fragments three web-ifc
```

**CRITICAL**: The Three.js version MUST match the version pinned by `@thatopen/components`. Run `npm ls three` and verify only ONE version is installed. Mismatched versions cause silent rendering failures.

### Complete Minimal Viewer

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
import * as THREE from "three";

// 1. Initialize component system
const components = new OBC.Components();

// 2. Create world (scene + camera + renderer)
const worlds = components.get(OBC.Worlds);
const world = worlds.create<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer>();

// 3. Attach to DOM
const container = document.getElementById("viewer")!;
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.SimpleCamera(components);
world.scene = new OBC.SimpleScene(components);

// 4. Configure IFC loader with WASM path
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: { path: "https://unpkg.com/web-ifc@0.0.77/", absolute: true },
});

// 5. Setup FragmentsManager — handles model lifecycle
const fragments = components.get(OBC.FragmentsManager);
fragments.list.onItemSet.add(({ value: model }) => {
  model.useCamera(world.camera.three);
  world.scene.three.add(model.object);
  fragments.core.update(true);
});

// 6. Add interaction components
const highlighter = components.get(OBCF.Highlighter);
highlighter.multiple = "ctrlKey";
highlighter.zoomToSelection = true;

const hider = components.get(OBC.Hider);

// 7. Load IFC file
const response = await fetch("/models/building.ifc");
const data = new Uint8Array(await response.arrayBuffer());
await ifcLoader.load(data, false, "my-model");
```

### Key Components Reference

| Component | Package | Purpose |
|-----------|---------|---------|
| `OBC.IfcLoader` | components | IFC → Fragments conversion and loading |
| `OBC.FragmentsManager` | components | Model lifecycle, Fragment caching, disposal |
| `OBC.Hider` | components | Hide, show, isolate elements by ID or category |
| `OBCF.Highlighter` | components-front | Selection highlighting via raycasting |
| `OBCF.Clipper` | components-front | Section planes through models |

### Selection and Property Inspection

```typescript
// Highlight on click — returns selected element IDs
const highlighter = components.get(OBCF.Highlighter);
await highlighter.highlight("select");

// Highlight by EXPRESS ID programmatically
const idMap: OBC.ModelIdMap = {};
idMap[model.modelId] = new Set([101, 102, 103]);
await highlighter.highlightByID("select", idMap);

// Clear selection
await highlighter.clear("select");
```

### Hide / Isolate Elements

```typescript
const hider = components.get(OBC.Hider);

// Isolate walls only
const wallIds = model.getItemsOfCategories(["IFCWALL"]);
const wallMap: OBC.ModelIdMap = {};
wallMap[model.modelId] = wallIds;
await hider.isolate(wallMap);

// Show everything again
await hider.set(true);
```

### Z-Fighting Prevention

ALWAYS apply polygon offset to Fragment materials to prevent z-fighting on coplanar BIM faces:

```typescript
fragments.core.models.materials.list.onItemSet.add(({ value: material }) => {
  material.polygonOffset = true;
  material.polygonOffsetUnits = 1;
  material.polygonOffsetFactor = Math.random();
});
```

---

## Option B: Custom web-ifc + Three.js Pipeline

Use this ONLY when integrating BIM data into an existing Three.js application where @thatopen/components cannot be adopted.

### Core Pipeline

```typescript
import * as THREE from "three";
import { IfcAPI } from "web-ifc";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";

// Initialize web-ifc
const ifcApi = new IfcAPI();
ifcApi.SetWasmPath("/static/wasm/");  // MUST be called BEFORE Init()
await ifcApi.Init();

// Load IFC file
const ifcData = new Uint8Array(await fetch("model.ifc").then(r => r.arrayBuffer()));
const modelID = ifcApi.OpenModel(ifcData, { COORDINATE_TO_ORIGIN: true });

// Store expressID → mesh mapping for selection
const expressIdToMesh = new Map<number, THREE.Mesh>();

// Stream geometry and convert to Three.js meshes
ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  const geometries = flatMesh.geometries;
  for (let i = 0; i < geometries.size(); i++) {
    const pg = geometries.get(i);
    const geom = ifcApi.GetGeometry(modelID, pg.geometryExpressID);

    // Vertex data: 6 floats per vertex (x, y, z, nx, ny, nz)
    const verts = ifcApi.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
    const idx = ifcApi.GetIndexArray(geom.GetIndexData(), geom.GetIndexDataSize());

    const positions = new Float32Array(verts.length / 2);
    const normals = new Float32Array(verts.length / 2);
    for (let j = 0; j < verts.length; j += 6) {
      const k = j / 2;
      positions[k] = verts[j]; positions[k+1] = verts[j+1]; positions[k+2] = verts[j+2];
      normals[k] = verts[j+3]; normals[k+1] = verts[j+4]; normals[k+2] = verts[j+5];
    }

    const bufGeom = new THREE.BufferGeometry();
    bufGeom.setAttribute("position", new THREE.BufferAttribute(positions, 3));
    bufGeom.setAttribute("normal", new THREE.BufferAttribute(normals, 3));
    bufGeom.setIndex(new THREE.BufferAttribute(idx, 1));
    bufGeom.applyMatrix4(new THREE.Matrix4().fromArray(pg.flatTransformation));

    const mesh = new THREE.Mesh(bufGeom, new THREE.MeshPhongMaterial({
      color: new THREE.Color(pg.color.x, pg.color.y, pg.color.z),
      opacity: pg.color.w,
      transparent: pg.color.w < 1.0,
      side: THREE.DoubleSide,
    }));
    mesh.userData.expressID = flatMesh.expressID;
    scene.add(mesh);
    expressIdToMesh.set(flatMesh.expressID, mesh);
  }
});
```

### Selection via Raycasting

```typescript
const raycaster = new THREE.Raycaster();
renderer.domElement.addEventListener("click", (event) => {
  const mouse = new THREE.Vector2(
    (event.clientX / window.innerWidth) * 2 - 1,
    -(event.clientY / window.innerHeight) * 2 + 1
  );
  raycaster.setFromCamera(mouse, camera);
  const hits = raycaster.intersectObjects(scene.children);
  if (hits.length > 0) {
    const expressID = hits[0].object.userData.expressID;
    const entity = ifcApi.GetLine(modelID, expressID, true);  // flatten=true
    displayProperties(entity);
  }
});
```

### Section Planes (Custom)

```typescript
const clipPlane = new THREE.Plane(new THREE.Vector3(0, -1, 0), 5);
renderer.clippingPlanes = [clipPlane];
renderer.localClippingEnabled = true;

// Update plane position interactively
function setClipHeight(y: number) {
  clipPlane.constant = y;
}
```

---

## Coordinate System Rules

| Source | Coordinate System | Action Required |
|--------|-------------------|-----------------|
| web-ifc geometry output | Y-up (transformed internally) | NONE — render directly |
| IfcOpenShell geometry | Z-up (IFC native) | ALWAYS rotate: `matrix.makeRotationX(-Math.PI / 2)` |
| `COORDINATE_TO_ORIGIN: true` | Centered at origin | Real-world coordinates are LOST — store separately if needed |
| `COORDINATE_TO_ORIGIN: false` | Real-world offset | Model may appear far from origin — camera setup required |

---

## Large Model Handling (>100MB IFC)

### Memory Constraints

| Constraint | Limit |
|-----------|-------|
| WASM linear memory | ~2GB practical maximum |
| Browser tab total | 2–4GB before crash |
| 8GB RAM laptop | ~1.7GB practical tab limit |

### Performance Strategy

1. **ALWAYS pre-convert to Fragments on server** for files >200MB. First-load IFC parsing is the bottleneck; cached Fragments load near-instantly.
2. **ALWAYS use `StreamAllMeshes`** instead of `LoadAllGeometry`. Streaming processes geometry incrementally without peak memory spike.
3. **ALWAYS call `ifcApi.CloseModel(modelID)`** when done to free WASM memory.
4. **Configure conservative memory limits**:

```typescript
const modelID = ifcApi.OpenModel(data, {
  MEMORY_LIMIT: 1073741824,  // 1GB — safe for most browsers
  TAPE_SIZE: 33554432,       // 32MB read buffer
});
```

5. **Use geometry instancing** for repeated elements (identical windows, doors). web-ifc provides shared `geometryExpressID` values that indicate reusable geometry.
6. **Export Fragments for caching**:

```typescript
const fragsBuffer = await model.getBuffer(false);
// Store on server or in IndexedDB for instant reload
```

---

## The Fragments Format

IFC files are NEVER rendered directly. The pipeline is:

```
IFC file → web-ifc WASM parsing → Fragment conversion → Three.js rendering
```

- First load: slow (parsing + tessellation + conversion)
- Subsequent loads from cached `.frag` files: near-instant
- Fragments use Google Flatbuffers — 5-10x smaller than source IFC
- ALWAYS cache Fragments server-side for production applications

---

## Property Access Patterns

| Operation | @thatopen/components | Raw web-ifc |
|-----------|---------------------|-------------|
| Get by type | `model.getItemsOfCategories(["IFCWALL"])` | `ifcApi.GetLineIDsWithType(modelID, IFCWALL)` |
| Get entity | Via FragmentsManager | `ifcApi.GetLine(modelID, expressID, true)` |
| Property sets | `ifcApi.properties.getPropertySets(modelID, id, true)` | Same |
| Spatial structure | `ifcApi.properties.getSpatialStructure(modelID)` | Same |
| Access name | `entity.Name.value` | `entity.Name.value` |

**CRITICAL**: web-ifc wraps string/numeric values in `{ value: ... }` containers. ALWAYS access `.value` for the actual data. References are `Handle` objects unless `flatten=true` is passed to `GetLine`.

---

## Critical Warnings

**NEVER** set the WASM path after calling `ifcApi.Init()` — the WASM module loads during `Init()` and path changes after that point have no effect.

**NEVER** use `LoadAllGeometry` for models with more than 10,000 elements — use `StreamAllMeshes` to avoid memory spikes.

**NEVER** mix Three.js versions — `@thatopen/components` pins a specific Three.js version. A second version in `node_modules` causes silent rendering failures.

**NEVER** manually rotate web-ifc output by -90 degrees — web-ifc ALREADY transforms IFC Z-up coordinates to Three.js Y-up internally.

**ALWAYS** call `CloseModel(modelID)` when disposing a model to free WASM memory.

**ALWAYS** verify the WASM path resolves correctly in your bundler (Webpack/Vite) — `.wasm` files MUST be copied to the output directory.

---

## Reference Links

- [references/methods.md](references/methods.md) — Complete API signatures for @thatopen/components and web-ifc viewer methods
- [references/examples.md](references/examples.md) — Working code examples for common viewer features
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do when building BIM web viewers

### Official Sources

- https://github.com/ThatOpen/engine_web-ifc (web-ifc v0.0.77)
- https://github.com/ThatOpen/engine_components (@thatopen/components v3.3.2)
- https://docs.thatopen.com/Tutorials/Components/Core/IfcLoader
- https://docs.thatopen.com/api/@thatopen/components-front/classes/Highlighter
- https://threejs.org/docs/
