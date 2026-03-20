# crosstech-impl-ifc-to-threejs — Examples

## Example 1: Minimal BIM Viewer with @thatopen/components

Complete working example for a browser-based BIM viewer with selection and property display.

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
import * as THREE from "three";

// 1. Component system initialization
const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer>();

// 2. Renderer, camera, scene
const container = document.getElementById("viewer")!;
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.SimpleCamera(components);
world.scene = new OBC.SimpleScene(components);

// 3. IFC loader — WASM path MUST be set before loading
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: { path: "https://unpkg.com/web-ifc@0.0.77/", absolute: true },
});

// 4. Fragment manager
const fragments = components.get(OBC.FragmentsManager);
fragments.list.onItemSet.add(({ value: model }) => {
  model.useCamera(world.camera.three);
  world.scene.three.add(model.object);
  fragments.core.update(true);
});

// 5. Z-fighting prevention
fragments.core.models.materials.list.onItemSet.add(({ value: material }) => {
  material.polygonOffset = true;
  material.polygonOffsetUnits = 1;
  material.polygonOffsetFactor = Math.random();
});

// 6. Highlighting for selection
const highlighter = components.get(OBCF.Highlighter);
highlighter.multiple = "ctrlKey";
highlighter.zoomToSelection = true;
highlighter.zoomFactor = 1.5;

// 7. Hider for visibility control
const hider = components.get(OBC.Hider);

// 8. Load IFC file
async function loadIfc(url: string): Promise<void> {
  const response = await fetch(url);
  const data = new Uint8Array(await response.arrayBuffer());
  await ifcLoader.load(data, false, "building");
}

await loadIfc("/models/office-building.ifc");
```

---

## Example 2: Custom web-ifc + Three.js Viewer with Property Panel

Full custom viewer with streaming geometry, raycasting selection, and property display.

```typescript
import * as THREE from "three";
import { IfcAPI } from "web-ifc";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";

// --- Three.js Scene Setup ---
const scene = new THREE.Scene();
scene.background = new THREE.Color(0xf0f0f0);

const camera = new THREE.PerspectiveCamera(
  60, window.innerWidth / window.innerHeight, 0.1, 1000
);
camera.position.set(20, 20, 20);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);

// Lighting
const ambient = new THREE.AmbientLight(0xffffff, 0.6);
scene.add(ambient);
const directional = new THREE.DirectionalLight(0xffffff, 0.8);
directional.position.set(50, 50, 50);
scene.add(directional);

// --- web-ifc Setup ---
const ifcApi = new IfcAPI();
ifcApi.SetWasmPath("/static/wasm/");
await ifcApi.Init();
ifcApi.SetLogLevel(0); // Silent

const ifcData = new Uint8Array(
  await fetch("/models/building.ifc").then(r => r.arrayBuffer())
);
const modelID = ifcApi.OpenModel(ifcData, {
  COORDINATE_TO_ORIGIN: true,
  MEMORY_LIMIT: 1073741824, // 1GB
});

// --- Geometry Streaming ---
const expressIdToMesh = new Map<number, THREE.Mesh>();

ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  const placedGeometries = flatMesh.geometries;
  for (let i = 0; i < placedGeometries.size(); i++) {
    const pg = placedGeometries.get(i);
    const geom = ifcApi.GetGeometry(modelID, pg.geometryExpressID);
    const verts = ifcApi.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
    const idx = ifcApi.GetIndexArray(geom.GetIndexData(), geom.GetIndexDataSize());

    // Deinterleave vertex data (6 floats: x,y,z,nx,ny,nz)
    const positions = new Float32Array(verts.length / 2);
    const normals = new Float32Array(verts.length / 2);
    for (let j = 0; j < verts.length; j += 6) {
      const k = j / 2;
      positions[k]     = verts[j];     positions[k + 1] = verts[j + 1]; positions[k + 2] = verts[j + 2];
      normals[k]       = verts[j + 3]; normals[k + 1]   = verts[j + 4]; normals[k + 2]   = verts[j + 5];
    }

    const bufGeom = new THREE.BufferGeometry();
    bufGeom.setAttribute("position", new THREE.BufferAttribute(positions, 3));
    bufGeom.setAttribute("normal", new THREE.BufferAttribute(normals, 3));
    bufGeom.setIndex(new THREE.BufferAttribute(idx, 1));

    const matrix = new THREE.Matrix4().fromArray(pg.flatTransformation);
    bufGeom.applyMatrix4(matrix);

    const material = new THREE.MeshPhongMaterial({
      color: new THREE.Color(pg.color.x, pg.color.y, pg.color.z),
      opacity: pg.color.w,
      transparent: pg.color.w < 1.0,
      side: THREE.DoubleSide,
      polygonOffset: true,
      polygonOffsetUnits: 1,
      polygonOffsetFactor: Math.random(),
    });

    const mesh = new THREE.Mesh(bufGeom, material);
    mesh.userData.expressID = flatMesh.expressID;
    scene.add(mesh);
    expressIdToMesh.set(flatMesh.expressID, mesh);
  }
});

// --- Raycasting Selection ---
const raycaster = new THREE.Raycaster();
let selectedMesh: THREE.Mesh | null = null;
const originalMaterials = new Map<THREE.Mesh, THREE.Material>();

renderer.domElement.addEventListener("click", async (event) => {
  // Restore previous selection
  if (selectedMesh && originalMaterials.has(selectedMesh)) {
    selectedMesh.material = originalMaterials.get(selectedMesh)!;
  }

  const mouse = new THREE.Vector2(
    (event.clientX / window.innerWidth) * 2 - 1,
    -(event.clientY / window.innerHeight) * 2 + 1
  );
  raycaster.setFromCamera(mouse, camera);
  const hits = raycaster.intersectObjects(scene.children, true);

  if (hits.length > 0) {
    const hitMesh = hits[0].object as THREE.Mesh;
    const expressID = hitMesh.userData.expressID;
    if (expressID === undefined) return;

    // Highlight selected mesh
    originalMaterials.set(hitMesh, hitMesh.material as THREE.Material);
    hitMesh.material = new THREE.MeshPhongMaterial({
      color: 0x00ff00,
      opacity: 0.8,
      transparent: true,
    });
    selectedMesh = hitMesh;

    // Fetch and display properties
    const entity = ifcApi.GetLine(modelID, expressID, true);
    const psets = await ifcApi.properties.getPropertySets(modelID, expressID, true);
    displayProperties(entity, psets);
  }
});

function displayProperties(entity: any, psets: any[]): void {
  const panel = document.getElementById("property-panel")!;
  let html = `<h3>${entity.Name?.value ?? "Unnamed"}</h3>`;
  html += `<p>Type: ${entity.constructor?.name ?? "Unknown"}</p>`;
  html += `<p>GlobalId: ${entity.GlobalId?.value ?? "N/A"}</p>`;

  for (const pset of psets) {
    if (!pset.HasProperties) continue;
    html += `<h4>${pset.Name?.value ?? "Property Set"}</h4><ul>`;
    for (const prop of pset.HasProperties) {
      if (prop.NominalValue) {
        html += `<li>${prop.Name.value}: ${prop.NominalValue.value}</li>`;
      }
    }
    html += "</ul>";
  }
  panel.innerHTML = html;
}

// --- Render Loop ---
function animate(): void {
  requestAnimationFrame(animate);
  controls.update();
  renderer.render(scene, camera);
}
animate();

// --- Cleanup on page unload ---
window.addEventListener("beforeunload", () => {
  ifcApi.CloseModel(modelID);
});
```

---

## Example 3: Storey-Based Visibility Toggle

Filter elements by building storey using spatial structure queries.

```typescript
// Using @thatopen/components
async function toggleStoreyVisibility(
  ifcApi: IfcAPI,
  modelID: number,
  model: any,
  hider: OBC.Hider,
  storeyName: string,
  visible: boolean
): Promise<void> {
  const structure = await ifcApi.properties.getSpatialStructure(modelID);

  function findStorey(node: any): any | null {
    if (node.type === "IFCBUILDINGSTOREY" && node.Name?.value === storeyName) {
      return node;
    }
    if (node.children) {
      for (const child of node.children) {
        const found = findStorey(child);
        if (found) return found;
      }
    }
    return null;
  }

  const storey = findStorey(structure);
  if (!storey) return;

  // Collect all expressIDs in this storey
  const ids = new Set<number>();
  function collectIds(node: any): void {
    ids.add(node.expressID);
    if (node.children) {
      for (const child of node.children) collectIds(child);
    }
  }
  collectIds(storey);

  const idMap: OBC.ModelIdMap = {};
  idMap[model.modelId] = ids;
  await hider.set(visible, idMap);
}
```

---

## Example 4: Server-Side Fragments Pre-Processing

Convert IFC to Fragments on the server for instant browser loading.

```typescript
// Server-side (Node.js)
import * as OBC from "@thatopen/components";
import * as fs from "fs";

const components = new OBC.Components();
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: { path: "./node_modules/web-ifc/", absolute: true },
});

const fragments = components.get(OBC.FragmentsManager);

// Convert IFC to Fragments
const ifcBuffer = new Uint8Array(fs.readFileSync("building.ifc"));
const model = await ifcLoader.load(ifcBuffer, false, "building");

// Export Fragments for browser delivery
const fragsBuffer = await model.getBuffer(false);
fs.writeFileSync("building.frag", Buffer.from(fragsBuffer));

// Client-side — load pre-converted Fragments (near-instant)
const fragData = new Uint8Array(await fetch("/models/building.frag").then(r => r.arrayBuffer()));
// Load via FragmentsManager instead of IfcLoader
```

---

## Example 5: Geometry Instancing for Repeated Elements

Reuse BufferGeometry for identical elements (windows, doors, furniture).

```typescript
// Track unique geometries by their geometry expressID
const geometryCache = new Map<number, THREE.BufferGeometry>();

ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  const placedGeometries = flatMesh.geometries;
  for (let i = 0; i < placedGeometries.size(); i++) {
    const pg = placedGeometries.get(i);

    // Check cache BEFORE extracting geometry
    let bufGeom = geometryCache.get(pg.geometryExpressID);
    if (!bufGeom) {
      const geom = ifcApi.GetGeometry(modelID, pg.geometryExpressID);
      const verts = ifcApi.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
      const idx = ifcApi.GetIndexArray(geom.GetIndexData(), geom.GetIndexDataSize());

      bufGeom = new THREE.BufferGeometry();
      const positions = new Float32Array(verts.length / 2);
      const normals = new Float32Array(verts.length / 2);
      for (let j = 0; j < verts.length; j += 6) {
        const k = j / 2;
        positions[k] = verts[j]; positions[k+1] = verts[j+1]; positions[k+2] = verts[j+2];
        normals[k] = verts[j+3]; normals[k+1] = verts[j+4]; normals[k+2] = verts[j+5];
      }
      bufGeom.setAttribute("position", new THREE.BufferAttribute(positions, 3));
      bufGeom.setAttribute("normal", new THREE.BufferAttribute(normals, 3));
      bufGeom.setIndex(new THREE.BufferAttribute(idx, 1));
      geometryCache.set(pg.geometryExpressID, bufGeom);
    }

    const material = new THREE.MeshPhongMaterial({
      color: new THREE.Color(pg.color.x, pg.color.y, pg.color.z),
      opacity: pg.color.w,
      transparent: pg.color.w < 1.0,
      side: THREE.DoubleSide,
    });

    const mesh = new THREE.Mesh(bufGeom.clone(), material);
    // Apply per-instance placement transform
    const matrix = new THREE.Matrix4().fromArray(pg.flatTransformation);
    mesh.applyMatrix4(matrix);
    mesh.userData.expressID = flatMesh.expressID;
    scene.add(mesh);
  }
});
```

---

## Example 6: Coordinate Handling — Z-up to Y-up

When receiving geometry from IfcOpenShell (server-side, Z-up) and rendering in Three.js (Y-up).

```typescript
// Server sends vertex positions in Z-up format (from IfcOpenShell)
const serverVertices: Float32Array = await fetchVerticesFromServer();

// Apply Z-up → Y-up rotation BEFORE creating BufferGeometry
const rotationMatrix = new THREE.Matrix4().makeRotationX(-Math.PI / 2);

const geometry = new THREE.BufferGeometry();
geometry.setAttribute("position", new THREE.BufferAttribute(serverVertices, 3));
geometry.applyMatrix4(rotationMatrix);

// When using web-ifc output: NO rotation needed — already Y-up
// ifcApi.StreamAllMeshes output is ALREADY in Y-up coordinates
```
