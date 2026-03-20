# Examples: IfcOpenShell and web-ifc Working Code

## Example 1: Load Model and Read All Wall Properties

### IfcOpenShell (Python)

```python
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("building.ifc")
print(f"Schema: {model.schema}")

walls = model.by_type("IfcWall")
for wall in walls:
    print(f"\nWall: {wall.Name} (ID: {wall.id()}, GUID: {wall.GlobalId})")

    # Get ALL properties (occurrence + type merged)
    psets = ifcopenshell.util.element.get_psets(wall)
    for pset_name, props in psets.items():
        print(f"  {pset_name}:")
        for prop_name, value in props.items():
            if prop_name != "id":  # Skip internal ID
                print(f"    {prop_name}: {value}")

    # Get spatial container
    container = ifcopenshell.util.element.get_container(wall)
    if container:
        print(f"  Container: {container.Name} ({container.is_a()})")

    # Get type
    wall_type = ifcopenshell.util.element.get_type(wall)
    if wall_type:
        print(f"  Type: {wall_type.Name}")
```

### web-ifc (JavaScript)

```javascript
import { IfcAPI, IFCWALL } from 'web-ifc';

const api = new IfcAPI();
await api.Init();

const response = await fetch('building.ifc');
const buffer = new Uint8Array(await response.arrayBuffer());
const modelID = api.OpenModel(buffer);

console.log(`Schema: ${api.GetModelSchema(modelID)}`);

const wallIDs = api.GetLineIDsWithType(modelID, IFCWALL);
for (let i = 0; i < wallIDs.size(); i++) {
  const id = wallIDs.get(i);
  const wall = api.GetLine(modelID, id, true);

  console.log(`\nWall: ${wall.Name?.value} (ID: ${id}, GUID: ${wall.GlobalId.value})`);

  // Get property sets via convenience API
  const psets = await api.properties.getPropertySets(modelID, id, true);
  for (const pset of psets) {
    console.log(`  ${pset.Name.value}:`);
    if (pset.HasProperties) {
      for (const prop of pset.HasProperties) {
        console.log(`    ${prop.Name.value}: ${prop.NominalValue?.value}`);
      }
    }
  }
}

// ALWAYS clean up
api.CloseModel(modelID);
```

---

## Example 2: Extract Geometry for a Single Element

### IfcOpenShell (Python)

```python
import ifcopenshell
import ifcopenshell.geom

model = ifcopenshell.open("building.ifc")
wall = model.by_type("IfcWall")[0]

settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

shape = ifcopenshell.geom.create_shape(settings, wall)

# Vertices: flat array [x1,y1,z1, x2,y2,z2, ...] -- Z-UP
verts = shape.geometry.verts
num_vertices = len(verts) // 3

# Faces: flat array [i1,i2,i3, ...] -- triangle indices
faces = shape.geometry.faces
num_triangles = len(faces) // 3

print(f"Vertices: {num_vertices}, Triangles: {num_triangles}")

# Access individual vertex (Z-UP coordinates)
for i in range(min(5, num_vertices)):
    x = verts[i * 3]
    y = verts[i * 3 + 1]
    z = verts[i * 3 + 2]
    print(f"  Vertex {i}: ({x:.3f}, {y:.3f}, {z:.3f})")
```

### web-ifc (JavaScript)

```javascript
import { IfcAPI, IFCWALL } from 'web-ifc';

const api = new IfcAPI();
await api.Init();
const modelID = api.OpenModel(data);

const wallIDs = api.GetLineIDsWithType(modelID, IFCWALL);
const wallID = wallIDs.get(0);

const flatMesh = api.GetFlatMesh(modelID, wallID);
const placedGeometries = flatMesh.geometries;

for (let i = 0; i < placedGeometries.size(); i++) {
  const pg = placedGeometries.get(i);
  const geom = api.GetGeometry(modelID, pg.geometryExpressID);

  // 6 floats per vertex: x, y, z, nx, ny, nz -- Y-UP
  const verts = api.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
  const indices = api.GetIndexArray(geom.GetIndexData(), geom.GetIndexDataSize());

  const numVertices = verts.length / 6;
  const numTriangles = indices.length / 3;
  console.log(`Geometry ${i}: Vertices: ${numVertices}, Triangles: ${numTriangles}`);

  // Access individual vertex (Y-UP coordinates)
  for (let j = 0; j < Math.min(5, numVertices); j++) {
    const x = verts[j * 6];
    const y = verts[j * 6 + 1];
    const z = verts[j * 6 + 2];
    console.log(`  Vertex ${j}: (${x.toFixed(3)}, ${y.toFixed(3)}, ${z.toFixed(3)})`);
  }

  // Placement transform
  const transform = pg.flatTransformation;
  console.log(`  Transform: [${Array.from(transform).map(v => v.toFixed(2)).join(', ')}]`);
}

api.CloseModel(modelID);
```

---

## Example 3: Stream All Geometry (Large Model)

### IfcOpenShell (Python)

```python
import multiprocessing
import ifcopenshell
import ifcopenshell.geom

model = ifcopenshell.open("large_model.ifc")
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

total_vertices = 0
total_triangles = 0
element_count = 0

iterator = ifcopenshell.geom.iterator(settings, model, multiprocessing.cpu_count())
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)

        verts = shape.geometry.verts
        faces = shape.geometry.faces
        total_vertices += len(verts) // 3
        total_triangles += len(faces) // 3
        element_count += 1

        if not iterator.next():
            break

print(f"Processed {element_count} elements")
print(f"Total vertices: {total_vertices}, Total triangles: {total_triangles}")
```

### web-ifc (JavaScript)

```javascript
import { IfcAPI } from 'web-ifc';

const api = new IfcAPI();
await api.Init();

const data = new Uint8Array(await fetch('large_model.ifc').then(r => r.arrayBuffer()));
const modelID = api.OpenModel(data, {
  COORDINATE_TO_ORIGIN: true,
  TAPE_SIZE: 134217728,  // 128MB for large files
});

let totalVertices = 0;
let totalTriangles = 0;
let elementCount = 0;

api.StreamAllMeshes(modelID, (flatMesh) => {
  const geoms = flatMesh.geometries;
  for (let i = 0; i < geoms.size(); i++) {
    const pg = geoms.get(i);
    const geom = api.GetGeometry(modelID, pg.geometryExpressID);
    const vertSize = geom.GetVertexDataSize();
    const idxSize = geom.GetIndexDataSize();

    totalVertices += vertSize / 6;
    totalTriangles += idxSize / 3;
  }
  elementCount++;
});

console.log(`Processed ${elementCount} elements`);
console.log(`Total vertices: ${totalVertices}, Total triangles: ${totalTriangles}`);

api.CloseModel(modelID);
```

---

## Example 4: Convert web-ifc Geometry to Three.js BufferGeometry

```javascript
import * as THREE from 'three';
import { IfcAPI } from 'web-ifc';

const api = new IfcAPI();
await api.Init();

const data = new Uint8Array(await fetch('model.ifc').then(r => r.arrayBuffer()));
const modelID = api.OpenModel(data);

const scene = new THREE.Scene();
const expressIDMap = new Map(); // Three.js mesh -> IFC expressID

api.StreamAllMeshes(modelID, (flatMesh) => {
  const placedGeometries = flatMesh.geometries;

  for (let i = 0; i < placedGeometries.size(); i++) {
    const pg = placedGeometries.get(i);
    const geom = api.GetGeometry(modelID, pg.geometryExpressID);

    const verts = api.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
    const indices = api.GetIndexArray(geom.GetIndexData(), geom.GetIndexDataSize());

    // Separate interleaved position + normal data
    const positions = new Float32Array(verts.length / 2);
    const normals = new Float32Array(verts.length / 2);
    for (let j = 0; j < verts.length; j += 6) {
      positions[j / 2]     = verts[j];
      positions[j / 2 + 1] = verts[j + 1];
      positions[j / 2 + 2] = verts[j + 2];
      normals[j / 2]       = verts[j + 3];
      normals[j / 2 + 1]   = verts[j + 4];
      normals[j / 2 + 2]   = verts[j + 5];
    }

    const bufferGeometry = new THREE.BufferGeometry();
    bufferGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    bufferGeometry.setAttribute('normal', new THREE.BufferAttribute(normals, 3));
    bufferGeometry.setIndex(new THREE.BufferAttribute(indices, 1));

    // Apply placement transform
    const matrix = new THREE.Matrix4().fromArray(pg.flatTransformation);
    bufferGeometry.applyMatrix4(matrix);

    // Material from IFC color
    const color = pg.color;
    const material = new THREE.MeshPhongMaterial({
      color: new THREE.Color(color.x, color.y, color.z),
      opacity: color.w,
      transparent: color.w < 1.0,
      side: THREE.DoubleSide,
    });

    const mesh = new THREE.Mesh(bufferGeometry, material);
    mesh.userData.expressID = flatMesh.expressID;
    scene.add(mesh);

    expressIDMap.set(mesh.uuid, flatMesh.expressID);
  }
});

api.CloseModel(modelID);
```

---

## Example 5: Hybrid Architecture -- Server Validation + Browser Viewing

### Server (Python / IfcOpenShell)

```python
from flask import Flask, jsonify, send_file
import ifcopenshell
import ifcopenshell.util.element

app = Flask(__name__)
model = None

@app.route('/api/load', methods=['POST'])
def load_model():
    global model
    model = ifcopenshell.open("uploaded_model.ifc")
    return jsonify({
        "schema": model.schema,
        "elements": len(model.by_type("IfcProduct")),
        "walls": len(model.by_type("IfcWall")),
    })

@app.route('/api/properties/<int:express_id>')
def get_properties(express_id):
    element = model.by_id(express_id)
    psets = ifcopenshell.util.element.get_psets(element)
    container = ifcopenshell.util.element.get_container(element)
    element_type = ifcopenshell.util.element.get_type(element)
    return jsonify({
        "name": element.Name,
        "globalId": element.GlobalId,
        "type": element.is_a(),
        "container": container.Name if container else None,
        "elementType": element_type.Name if element_type else None,
        "propertySets": psets,
    })

@app.route('/api/model.ifc')
def serve_model():
    return send_file("uploaded_model.ifc", mimetype="application/octet-stream")
```

### Browser (JavaScript / web-ifc)

```javascript
import { IfcAPI } from 'web-ifc';

const api = new IfcAPI();
await api.Init();

// Load model from server for visualization
const response = await fetch('/api/model.ifc');
const data = new Uint8Array(await response.arrayBuffer());
const modelID = api.OpenModel(data);

// Use web-ifc for fast geometry streaming
api.StreamAllMeshes(modelID, (flatMesh) => {
  // Build Three.js scene (see Example 4)
});

// Use server for rich property queries (IfcOpenShell has better utilities)
async function getElementDetails(expressID) {
  const response = await fetch(`/api/properties/${expressID}`);
  return response.json();
}

// On element click in Three.js viewer:
async function onElementSelected(expressID) {
  // Quick properties from web-ifc (browser-side)
  const basicProps = api.GetLine(modelID, expressID, true);
  console.log(`Selected: ${basicProps.Name?.value}`);

  // Rich properties from IfcOpenShell (server-side)
  const details = await getElementDetails(expressID);
  console.log(`Container: ${details.container}`);
  console.log(`Type: ${details.elementType}`);
  console.log(`Properties:`, details.propertySets);
}
```

---

## Example 6: Schema Version Check on Both Sides

### IfcOpenShell (Python)

```python
import ifcopenshell

model = ifcopenshell.open("model.ifc")
schema = model.schema  # "IFC2X3", "IFC4", "IFC4X3"

if schema == "IFC4X3":
    alignments = model.by_type("IfcAlignment")
    roads = model.by_type("IfcRoad")
    print(f"Infrastructure model: {len(alignments)} alignments, {len(roads)} roads")
elif schema == "IFC4":
    # Standard building model
    pass
elif schema == "IFC2X3":
    # Legacy model -- IfcWallStandardCase exists here, not in IFC4+
    pass
```

### web-ifc (JavaScript)

```javascript
import { IfcAPI, Schemas } from 'web-ifc';

const api = new IfcAPI();
await api.Init();
const modelID = api.OpenModel(data);

const schema = api.GetModelSchema(modelID);
console.log(`Schema: ${schema}`);

if (schema === 'IFC4X3') {
  // Infrastructure entities available
} else if (schema === 'IFC4') {
  // Standard building model
} else if (schema === 'IFC2X3') {
  // Legacy model
}
```
