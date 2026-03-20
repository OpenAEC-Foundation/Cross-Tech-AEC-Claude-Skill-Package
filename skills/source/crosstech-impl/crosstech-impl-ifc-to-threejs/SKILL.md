---
name: crosstech-impl-ifc-to-threejs
description: >
  Use when loading IFC models into Three.js scenes for web-based BIM visualization.
  Prevents the common mistake of losing IFC metadata in the Three.js scene graph
  or ignoring the Y-up vs Z-up axis convention difference.
  Covers web-ifc-three, @thatopen/components, mesh conversion, material mapping,
  raycasting for selection, property panels, and the Fragments optimization format.
  Keywords: IFC, Three.js, web viewer, BIM visualization, web-ifc-three,
  @thatopen/components, raycasting, scene graph, IFC viewer, WebGL.
license: MIT
compatibility: "Designed for Claude Code. Covers Three.js r160+, web-ifc 0.0.77, @thatopen/components 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-impl-ifc-to-threejs

## Quick Reference

### Technology Boundary

| Aspect | Side A: IFC / web-ifc | Side B: Three.js |
|--------|----------------------|------------------|
| Data model | IFC entities with properties, relationships, geometry | Scene graph: Object3D, Mesh, BufferGeometry |
| Coordinate system | Z-up (Z vertical, X/Y ground plane) | Y-up (Y vertical, X/Z ground plane) |
| Geometry format | Parametric (extruded solids, BREP, CSG) | Tessellated triangles (BufferGeometry) |
| Properties | IfcPropertySet, IfcElementQuantity via relationships | NOT stored in scene graph — userData only |
| Materials | IfcSurfaceStyle, IfcMaterialLayerSetUsage | MeshPhongMaterial, MeshStandardMaterial |
| Identifiers | expressID (integer per entity) | Object3D.userData or Fragment index |
| Directionality | **IFC → Three.js (one-way rendering)** | No write-back to IFC |

### The Bridge: Two Approaches

| Approach | Package | When to Use |
|----------|---------|-------------|
| **@thatopen/components** (recommended) | `@thatopen/components` + `@thatopen/components-front` | Production BIM viewers, rapid development |
| **Custom web-ifc + Three.js** | `web-ifc` + `three` | Full rendering control, existing Three.js apps |

### Critical Warnings

**NEVER** assume IFC properties exist in the Three.js scene graph. Properties are stored in the IFC model and MUST be queried via web-ifc or @thatopen/components APIs separately.

**NEVER** manually rotate the model by -90° around X when using web-ifc. web-ifc ALREADY transforms geometry from Z-up to Y-up internally. Applying a second rotation produces upside-down models.

**NEVER** use `LoadAllGeometry` for production models. It loads ALL geometry into memory at once. ALWAYS use `StreamAllMeshes` for incremental processing.

**NEVER** use a Three.js version different from the one pinned by @thatopen/components. Version mismatches cause silent rendering failures. ALWAYS verify with `npm ls three`.

**NEVER** forget to call `ifcApi.CloseModel(modelID)` when done. WASM memory is NOT garbage collected by JavaScript.

**NEVER** set the WASM path AFTER calling `ifcApi.Init()`. The path MUST be configured BEFORE initialization.

---

## Technology Boundary Detail

### Side A: IFC / web-ifc

IFC files contain parametric geometry, semantic properties, spatial hierarchy, and material definitions. web-ifc (v0.0.77) parses IFC via WebAssembly and provides:

- **Model loading**: `IfcAPI.OpenModel(data, settings)` returns a modelID
- **Geometry extraction**: `StreamAllMeshes` yields `FlatMesh` objects with vertex/index arrays
- **Property access**: `ifcApi.properties.getPropertySets(modelID, expressID)` or manual `GetLine`
- **Spatial structure**: `ifcApi.properties.getSpatialStructure(modelID)` returns the full hierarchy
- **Schema detection**: `ifcApi.GetModelSchema(modelID)` returns "IFC2X3", "IFC4", or "IFC4X3"

Vertex data format from web-ifc: 6 floats per vertex (x, y, z, nx, ny, nz — position + normal interleaved).

### Side B: Three.js Scene

Three.js (r160+) renders triangulated meshes via WebGL/WebGPU:

- **BufferGeometry**: position attribute (Float32Array, 3 components), normal attribute (Float32Array, 3 components), index (Uint32Array)
- **Materials**: MeshPhongMaterial or MeshStandardMaterial with color, opacity, transparency
- **Scene graph**: Object3D hierarchy for spatial organization
- **Raycasting**: Raycaster intersects meshes for element selection
- **Camera**: PerspectiveCamera with OrbitControls for BIM navigation

### The Bridge: Conversion Pipeline

```
IFC File (.ifc)
    │
    ▼
web-ifc (WASM) ─── parses IFC, tessellates geometry, transforms Z-up → Y-up
    │
    ├── Option A: @thatopen/components
    │   IFC → Fragments (Flatbuffers) → Three.js Scene
    │   Automatic: highlighting, hiding, clipping, properties
    │
    └── Option B: Custom pipeline
        StreamAllMeshes → BufferGeometry + Material → Three.js Scene
        Manual: raycasting, property panels, spatial hierarchy
```

---

## Axis Convention Handling

### Rule: web-ifc Handles the Transform

web-ifc converts IFC Z-up coordinates to Three.js Y-up coordinates internally during geometry extraction. The output of `GetVertexArray` is ALREADY in Y-up format.

**When you MUST apply a manual transform**:
- Receiving coordinates from IfcOpenShell (server-side, Z-up output)
- Reading raw IFC placement matrices without web-ifc processing
- Importing point clouds or external meshes that use Z-up

```typescript
// ONLY for Z-up data NOT processed by web-ifc
const rotationMatrix = new THREE.Matrix4().makeRotationX(-Math.PI / 2);
geometry.applyMatrix4(rotationMatrix);
```

### COORDINATE_TO_ORIGIN Setting

web-ifc defaults `COORDINATE_TO_ORIGIN: true`, which centers geometry at the world origin. This removes real-world coordinates (easting/northing). If you need georeferenced positioning:

```typescript
const modelID = ifcApi.OpenModel(data, {
  COORDINATE_TO_ORIGIN: false, // Keep original IFC coordinates
});
```

---

## Essential Patterns

### Pattern 1: @thatopen/components BIM Viewer (Recommended)

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
import * as THREE from "three";

const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer>();

const container = document.getElementById("viewer")!;
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.SimpleCamera(components);
world.scene = new OBC.SimpleScene(components);

// IFC loader with WASM configuration
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: { path: "https://unpkg.com/web-ifc@0.0.77/", absolute: true },
});

// Fragment manager — handles model lifecycle
const fragments = components.get(OBC.FragmentsManager);
fragments.list.onItemSet.add(({ value: model }) => {
  model.useCamera(world.camera.three);
  world.scene.three.add(model.object);
  fragments.core.update(true);
});

// Load IFC
const response = await fetch("/models/building.ifc");
const data = new Uint8Array(await response.arrayBuffer());
await ifcLoader.load(data, false, "building");

// Interaction: highlighting + hiding
const highlighter = components.get(OBCF.Highlighter);
highlighter.multiple = "ctrlKey";
highlighter.zoomToSelection = true;

const hider = components.get(OBC.Hider);
```

### Pattern 2: Custom web-ifc + Three.js Pipeline

```typescript
import * as THREE from "three";
import { IfcAPI } from "web-ifc";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";

// Initialize web-ifc
const ifcApi = new IfcAPI();
ifcApi.SetWasmPath("/static/wasm/"); // BEFORE Init()
await ifcApi.Init();

// Load model
const ifcData = new Uint8Array(await fetch("model.ifc").then(r => r.arrayBuffer()));
const modelID = ifcApi.OpenModel(ifcData, { COORDINATE_TO_ORIGIN: true });

// expressID → mesh mapping for selection
const expressIdToMesh = new Map<number, THREE.Mesh>();

// Stream geometry and convert to Three.js
ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  const placedGeometries = flatMesh.geometries;
  for (let i = 0; i < placedGeometries.size(); i++) {
    const pg = placedGeometries.get(i);
    const geom = ifcApi.GetGeometry(modelID, pg.geometryExpressID);
    const verts = ifcApi.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
    const idx = ifcApi.GetIndexArray(geom.GetIndexData(), geom.GetIndexDataSize());

    // Deinterleave: 6 floats per vertex → separate position and normal
    const positions = new Float32Array(verts.length / 2);
    const normals = new Float32Array(verts.length / 2);
    for (let j = 0; j < verts.length; j += 6) {
      const k = j / 2;
      positions[k] = verts[j];     positions[k+1] = verts[j+1]; positions[k+2] = verts[j+2];
      normals[k]   = verts[j+3];   normals[k+1]   = verts[j+4]; normals[k+2]   = verts[j+5];
    }

    const bufGeom = new THREE.BufferGeometry();
    bufGeom.setAttribute("position", new THREE.BufferAttribute(positions, 3));
    bufGeom.setAttribute("normal", new THREE.BufferAttribute(normals, 3));
    bufGeom.setIndex(new THREE.BufferAttribute(idx, 1));

    // Apply IFC placement transform
    const matrix = new THREE.Matrix4().fromArray(pg.flatTransformation);
    bufGeom.applyMatrix4(matrix);

    // Material from IFC surface color
    const color = pg.color;
    const material = new THREE.MeshPhongMaterial({
      color: new THREE.Color(color.x, color.y, color.z),
      opacity: color.w,
      transparent: color.w < 1.0,
      side: THREE.DoubleSide,
    });

    const mesh = new THREE.Mesh(bufGeom, material);
    mesh.userData.expressID = flatMesh.expressID;
    scene.add(mesh);
    expressIdToMesh.set(flatMesh.expressID, mesh);
  }
});
```

### Pattern 3: Raycasting for Element Selection

```typescript
const raycaster = new THREE.Raycaster();

renderer.domElement.addEventListener("click", async (event) => {
  const mouse = new THREE.Vector2(
    (event.clientX / window.innerWidth) * 2 - 1,
    -(event.clientY / window.innerHeight) * 2 + 1
  );
  raycaster.setFromCamera(mouse, camera);
  const hits = raycaster.intersectObjects(scene.children, true);

  if (hits.length > 0) {
    const expressID = hits[0].object.userData.expressID;
    if (expressID === undefined) return;

    // Get entity info
    const entity = ifcApi.GetLine(modelID, expressID, true);
    console.log("Selected:", entity.Name?.value, "Type:", entity.constructor.name);

    // Get property sets
    const psets = await ifcApi.properties.getPropertySets(modelID, expressID, true);
    displayPropertyPanel(entity, psets);
  }
});
```

### Pattern 4: Scene Graph Mirroring IFC Spatial Structure

```typescript
async function buildSpatialHierarchy(
  ifcApi: IfcAPI, modelID: number, scene: THREE.Scene
): Promise<Map<number, THREE.Group>> {
  const structure = await ifcApi.properties.getSpatialStructure(modelID);
  const groups = new Map<number, THREE.Group>();

  function traverse(node: any, parent: THREE.Object3D) {
    const group = new THREE.Group();
    group.name = node.Name?.value ?? `Element_${node.expressID}`;
    group.userData.expressID = node.expressID;
    parent.add(group);
    groups.set(node.expressID, group);

    if (node.children) {
      for (const child of node.children) {
        traverse(child, group);
      }
    }
  }

  traverse(structure, scene);
  return groups;
}
```

---

## Fragments Format

@thatopen/components does NOT render IFC geometry directly. The pipeline is:

1. **IFC file** → web-ifc parses and tessellates
2. **Fragments conversion** → Flatbuffers-based binary format optimized for GPU
3. **Three.js rendering** → Fragments are rendered as instanced meshes

**Key implications**:
- First load is slow (parse + tessellate + convert)
- Exported `.frag` files load near-instantly on subsequent visits
- Server-side pre-conversion eliminates client-side processing cost

```typescript
// Export model to .frag for caching
const fragsBuffer = await model.getBuffer(false);
const file = new File([fragsBuffer], "model.frag");
// Upload to server or store in IndexedDB
```

---

## Common Operations

### Property Panel for Selected Element

```typescript
async function getElementProperties(
  ifcApi: IfcAPI, modelID: number, expressID: number
): Promise<Record<string, Record<string, unknown>>> {
  const result: Record<string, Record<string, unknown>> = {};

  const psets = await ifcApi.properties.getPropertySets(modelID, expressID, true);
  for (const pset of psets) {
    if (!pset.HasProperties) continue;
    const setName = pset.Name?.value ?? "Unnamed";
    result[setName] = {};
    for (const prop of pset.HasProperties) {
      if (prop.NominalValue) {
        result[setName][prop.Name.value] = prop.NominalValue.value;
      }
    }
  }
  return result;
}
```

### Category-Based Visibility (Hider Pattern)

```typescript
// @thatopen/components approach
const wallIds = model.getItemsOfCategories(["IFCWALL"]);
const wallMap: OBC.ModelIdMap = {};
wallMap[model.modelId] = wallIds;
await hider.isolate(wallMap); // Show ONLY walls

// Custom approach
function hideByType(ifcApi: IfcAPI, modelID: number, typeCode: number,
                    meshMap: Map<number, THREE.Mesh>, visible: boolean) {
  const ids = ifcApi.GetLineIDsWithType(modelID, typeCode);
  for (let i = 0; i < ids.size(); i++) {
    const mesh = meshMap.get(ids.get(i));
    if (mesh) mesh.visible = visible;
  }
}
```

### Z-Fighting Prevention

BIM models contain many coplanar surfaces (walls meeting slabs, stacked floors). ALWAYS enable polygon offset:

```typescript
// @thatopen/components
fragments.core.models.materials.list.onItemSet.add(({ value: material }) => {
  material.polygonOffset = true;
  material.polygonOffsetUnits = 1;
  material.polygonOffsetFactor = Math.random();
});

// Custom pipeline — apply to every material
const material = new THREE.MeshPhongMaterial({ /* ... */ });
material.polygonOffset = true;
material.polygonOffsetUnits = 1;
material.polygonOffsetFactor = Math.random();
```

### Memory Management

```typescript
// ALWAYS set memory limits for production
const modelID = ifcApi.OpenModel(data, {
  MEMORY_LIMIT: 1073741824,  // 1GB — conservative for browsers
  TAPE_SIZE: 33554432,       // 32MB read buffer
});

// ALWAYS close models when done
ifcApi.CloseModel(modelID);

// For @thatopen/components — dispose fragments
fragments.dispose();
```

---

## Decision Tree

```
START: You need IFC in a Three.js scene
│
├── Q1: Building a new BIM viewer from scratch?
│   ├── YES → Use @thatopen/components (Pattern 1)
│   │   Provides: loading, highlighting, hiding, clipping, properties
│   └── NO → Integrating into existing Three.js app?
│       ├── YES → Use custom web-ifc pipeline (Pattern 2)
│       └── NO → Just need static 3D preview?
│           └── Use @thatopen/components with minimal config
│
├── Q2: Do you need IFC property access?
│   ├── YES → MUST maintain web-ifc model reference alongside Three.js scene
│   │   Properties are NOT in the scene graph — query via ifcApi or components API
│   └── NO → Geometry-only conversion is sufficient
│
├── Q3: Model size > 100MB?
│   ├── YES → Pre-convert to Fragments on server, serve .frag files
│   │   Use StreamAllMeshes (NEVER LoadAllGeometry)
│   │   Set MEMORY_LIMIT in LoaderSettings
│   └── NO → Direct browser loading is acceptable
│
├── Q4: Coordinates from IfcOpenShell (server-side)?
│   ├── YES → Apply rotation: Matrix4.makeRotationX(-Math.PI / 2)
│   └── NO (from web-ifc) → No rotation needed, already Y-up
│
└── Q5: Need georeferenced coordinates?
    ├── YES → Set COORDINATE_TO_ORIGIN: false
    └── NO → Keep default (true) for centered viewing
```

---

## Data Loss Summary

| IFC Data | Three.js Representation | Status |
|----------|------------------------|--------|
| Geometry (parametric) | Tessellated BufferGeometry | **LOST** — triangulation is irreversible |
| Surface colors | MeshPhongMaterial / MeshStandardMaterial | Simplified |
| Material layers | Single material per mesh | **LOST** — layer structure not preserved |
| Properties (Psets) | NOT in scene graph | Available via API only |
| Quantities | NOT in scene graph | Available via API only |
| Spatial hierarchy | Optional Group hierarchy | Partial — requires manual setup |
| Relationships | NOT represented | **LOST** unless queried via API |
| GlobalId | userData or Fragment index | Available via API only |
| Type definitions | NOT in scene graph | Available via API only |

---

## Reference Links

- [references/methods.md](references/methods.md) — API signatures for web-ifc geometry, @thatopen/components, Three.js conversion
- [references/examples.md](references/examples.md) — Complete working examples for both approaches
- [references/anti-patterns.md](references/anti-patterns.md) — Common mistakes in IFC→Three.js integration

### Official Sources

- https://github.com/ThatOpen/engine_web-ifc — web-ifc source and API (v0.0.77)
- https://github.com/ThatOpen/engine_components — @thatopen/components (v3.3.2)
- https://docs.thatopen.com/ — ThatOpen documentation
- https://threejs.org/docs/ — Three.js documentation
- https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/ — IFC4 schema
