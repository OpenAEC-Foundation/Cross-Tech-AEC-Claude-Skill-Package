# crosstech-impl-bim-web-viewer — Examples

## Example 1: Complete @thatopen/components Viewer with All Features

A production-ready BIM viewer with navigation, selection, property panel, hiding, and clipping.

```typescript
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
import * as THREE from "three";

// --- Initialization ---
const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer>();

const container = document.getElementById("viewer")!;
world.renderer = new OBC.SimpleRenderer(components, container);
world.camera = new OBC.SimpleCamera(components);
world.scene = new OBC.SimpleScene(components);

// --- IFC Loader ---
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup({
  autoSetWasm: false,
  wasm: { path: "https://unpkg.com/web-ifc@0.0.77/", absolute: true },
});

// --- Fragments Manager with z-fighting fix ---
const fragments = components.get(OBC.FragmentsManager);
fragments.core.models.materials.list.onItemSet.add(({ value: material }) => {
  material.polygonOffset = true;
  material.polygonOffsetUnits = 1;
  material.polygonOffsetFactor = Math.random();
});

fragments.list.onItemSet.add(({ value: model }) => {
  model.useCamera(world.camera.three);
  world.scene.three.add(model.object);
  fragments.core.update(true);
});

// --- Highlighter (selection) ---
const highlighter = components.get(OBCF.Highlighter);
highlighter.multiple = "ctrlKey";
highlighter.zoomToSelection = true;
highlighter.zoomFactor = 1.5;

// --- Hider (visibility) ---
const hider = components.get(OBC.Hider);

// --- Load model ---
const response = await fetch("/models/office-building.ifc");
const data = new Uint8Array(await response.arrayBuffer());
const model = await ifcLoader.load(data, false, "office-building");

// --- Isolate walls ---
function isolateWalls() {
  const wallIds = model.getItemsOfCategories(["IFCWALL"]);
  const wallMap: OBC.ModelIdMap = {};
  wallMap[model.modelId] = wallIds;
  hider.isolate(wallMap);
}

// --- Show all elements ---
function showAll() {
  hider.set(true);
}

// --- Highlight specific elements ---
function selectElements(expressIds: number[]) {
  const idMap: OBC.ModelIdMap = {};
  idMap[model.modelId] = new Set(expressIds);
  highlighter.highlightByID("select", idMap);
}
```

## Example 2: Property Panel from Selection

Retrieve and display IFC properties when a user selects an element.

```typescript
import { IfcAPI } from "web-ifc";

// Assume ifcApi is initialized and model is loaded
const ifcApi = new IfcAPI();
await ifcApi.Init();

async function showProperties(modelID: number, expressID: number): Promise<void> {
  // Get basic entity data (flatten=true resolves all references)
  const entity = ifcApi.GetLine(modelID, expressID, true);

  const panel = document.getElementById("property-panel")!;
  panel.innerHTML = "";

  // Basic info
  addRow(panel, "Name", entity.Name?.value ?? "Unnamed");
  addRow(panel, "Type", entity.constructor?.name ?? "Unknown");
  addRow(panel, "GlobalId", entity.GlobalId?.value ?? "");

  // Property sets
  const psets = await ifcApi.properties.getPropertySets(modelID, expressID, true);
  for (const pset of psets) {
    const groupTitle = pset.Name?.value ?? "Properties";
    addGroupHeader(panel, groupTitle);

    if (pset.HasProperties) {
      for (const prop of pset.HasProperties) {
        const name = prop.Name?.value ?? "";
        const value = prop.NominalValue?.value ?? "";
        addRow(panel, name, String(value));
      }
    }
  }

  // Materials
  const materials = await ifcApi.properties.getMaterialsProperties(modelID, expressID);
  if (materials.length > 0) {
    addGroupHeader(panel, "Materials");
    for (const mat of materials) {
      addRow(panel, "Material", mat.Name?.value ?? "Unnamed");
    }
  }
}

function addRow(panel: HTMLElement, label: string, value: string): void {
  const row = document.createElement("div");
  row.innerHTML = `<strong>${label}:</strong> ${value}`;
  panel.appendChild(row);
}

function addGroupHeader(panel: HTMLElement, title: string): void {
  const header = document.createElement("h3");
  header.textContent = title;
  panel.appendChild(header);
}
```

## Example 3: Custom Section Plane with Three.js

Manual clipping plane implementation for a custom viewer (Option B).

```typescript
import * as THREE from "three";

// Setup renderer with clipping support
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.localClippingEnabled = true;

// Create a horizontal clip plane (cuts from above)
const clipPlane = new THREE.Plane(new THREE.Vector3(0, -1, 0), 10);

// Apply to all materials in the scene
scene.traverse((child) => {
  if (child instanceof THREE.Mesh && child.material) {
    const mat = child.material as THREE.MeshPhongMaterial;
    mat.clippingPlanes = [clipPlane];
    mat.clipShadows = true;
  }
});

// Interactive slider to adjust clip height
const slider = document.getElementById("clip-slider") as HTMLInputElement;
slider.addEventListener("input", () => {
  clipPlane.constant = parseFloat(slider.value);
});

// Visual helper to show clip plane position
const planeHelper = new THREE.PlaneHelper(clipPlane, 50, 0xff0000);
scene.add(planeHelper);
```

## Example 4: Storey Navigation

Filter model by building storey using the IFC spatial structure.

```typescript
// Get spatial structure from web-ifc
const structure = await ifcApi.properties.getSpatialStructure(modelID);

// Structure is: IfcProject → IfcSite → IfcBuilding → IfcBuildingStorey[]
function getStoreys(structure: any): Array<{ name: string; expressID: number }> {
  const storeys: Array<{ name: string; expressID: number }> = [];

  function traverse(node: any) {
    if (node.type === "IFCBUILDINGSTOREY") {
      storeys.push({
        name: node.Name?.value ?? `Storey ${node.expressID}`,
        expressID: node.expressID,
      });
    }
    if (node.children) {
      for (const child of node.children) {
        traverse(child);
      }
    }
  }

  traverse(structure);
  return storeys;
}

// Build storey selector UI
const storeys = getStoreys(structure);
const selector = document.getElementById("storey-select") as HTMLSelectElement;

for (const storey of storeys) {
  const option = document.createElement("option");
  option.value = String(storey.expressID);
  option.textContent = storey.name;
  selector.appendChild(option);
}

// On selection: isolate that storey's elements
selector.addEventListener("change", async () => {
  const storeyId = parseInt(selector.value);
  // Get all elements contained in this storey
  const storeyEntity = ifcApi.GetLine(modelID, storeyId, true);
  // Use Hider to isolate (with @thatopen/components)
  // or set mesh.visible for custom viewer
});
```

## Example 5: Caching Fragments for Instant Reload

Pre-convert IFC to Fragments and cache for subsequent visits.

```typescript
const CACHE_KEY = "model-fragments";

async function loadModel(ifcUrl: string): Promise<void> {
  // Check cache first
  const cached = await loadFromIndexedDB(CACHE_KEY);

  if (cached) {
    // Instant load from cached Fragments
    await fragments.load(cached);
    console.log("Loaded from cache — instant");
    return;
  }

  // First load: parse IFC and convert to Fragments
  const response = await fetch(ifcUrl);
  const data = new Uint8Array(await response.arrayBuffer());
  const model = await ifcLoader.load(data, false, "cached-model");

  // Cache the Fragments for next time
  const fragsBuffer = await model.getBuffer(false);
  await saveToIndexedDB(CACHE_KEY, fragsBuffer);
  console.log("Converted and cached for next visit");
}

// IndexedDB helpers (simplified)
function saveToIndexedDB(key: string, data: ArrayBuffer): Promise<void> {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open("bim-cache", 1);
    request.onupgradeneeded = () => {
      request.result.createObjectStore("fragments");
    };
    request.onsuccess = () => {
      const tx = request.result.transaction("fragments", "readwrite");
      tx.objectStore("fragments").put(data, key);
      tx.oncomplete = () => resolve();
      tx.onerror = () => reject(tx.error);
    };
  });
}

function loadFromIndexedDB(key: string): Promise<ArrayBuffer | null> {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open("bim-cache", 1);
    request.onupgradeneeded = () => {
      request.result.createObjectStore("fragments");
    };
    request.onsuccess = () => {
      const tx = request.result.transaction("fragments", "readonly");
      const getReq = tx.objectStore("fragments").get(key);
      getReq.onsuccess = () => resolve(getReq.result ?? null);
      getReq.onerror = () => reject(getReq.error);
    };
  });
}
```

## Example 6: Large Model with Memory-Safe Settings

Loading a 300MB IFC file with conservative memory settings and streaming.

```typescript
import { IfcAPI, LogLevel } from "web-ifc";

const ifcApi = new IfcAPI();
ifcApi.SetWasmPath("/static/wasm/");
await ifcApi.Init();
ifcApi.SetLogLevel(LogLevel.LOG_LEVEL_ERROR);

// Fetch with progress tracking
const response = await fetch("/models/large-hospital.ifc");
const reader = response.body!.getReader();
const contentLength = parseInt(response.headers.get("Content-Length") ?? "0");

let receivedLength = 0;
const chunks: Uint8Array[] = [];
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  chunks.push(value);
  receivedLength += value.length;
  console.log(`Download: ${Math.round(receivedLength / contentLength * 100)}%`);
}

// Combine chunks
const data = new Uint8Array(receivedLength);
let offset = 0;
for (const chunk of chunks) {
  data.set(chunk, offset);
  offset += chunk.length;
}

// Open with conservative settings
const modelID = ifcApi.OpenModel(data, {
  COORDINATE_TO_ORIGIN: true,
  MEMORY_LIMIT: 1073741824,  // 1GB
  TAPE_SIZE: 33554432,       // 32MB
  CIRCLE_SEGMENTS: 12,       // Reduce tessellation for performance
});

// Stream geometry — NEVER use LoadAllGeometry for large models
let meshCount = 0;
ifcApi.StreamAllMeshes(modelID, (flatMesh) => {
  meshCount++;
  if (meshCount % 1000 === 0) {
    console.log(`Processed ${meshCount} meshes`);
  }
  // Convert to Three.js geometry (see Example in SKILL.md Option B)
});

console.log(`Total: ${meshCount} meshes loaded`);

// ALWAYS clean up when done
// ifcApi.CloseModel(modelID);
```
