# crosstech-impl-speckle-blender — Methods Reference

## SpecklePy Core API

### Authentication

```python
from specklepy.api.client import SpeckleClient
from specklepy.api.credentials import get_default_account

# Connect and authenticate
client = SpeckleClient(host="https://app.speckle.systems")
account = get_default_account()
client.authenticate_with_account(account)

# Token-based alternative
client.authenticate_with_token("YOUR_PERSONAL_ACCESS_TOKEN")
```

### Stream/Project Operations

| Method | Signature | Returns |
|--------|-----------|---------|
| Create stream | `client.stream.create(name: str, description: str = "", is_public: bool = True)` | `str` (stream_id) |
| Get stream | `client.stream.get(id: str)` | `Stream` |
| List streams | `client.stream.list(stream_limit: int = 10)` | `List[Stream]` |
| Search streams | `client.stream.search(search_query: str)` | `List[Stream]` |

### Branch/Model Operations

| Method | Signature | Returns |
|--------|-----------|---------|
| List branches | `client.branch.list(stream_id: str, branches_limit: int = 10)` | `List[Branch]` |
| Create branch | `client.branch.create(stream_id: str, name: str, description: str = "")` | `str` (branch_id) |
| Get branch | `client.branch.get(stream_id: str, name: str)` | `Branch` |

### Commit/Version Operations

| Method | Signature | Returns |
|--------|-----------|---------|
| List commits | `client.commit.list(stream_id: str, limit: int = 10)` | `List[Commit]` |
| Get commit | `client.commit.get(stream_id: str, commit_id: str)` | `Commit` |
| Create commit | `client.commit.create(stream_id: str, object_id: str, branch_name: str = "main", message: str = "")` | `str` (commit_id) |

### Send/Receive Operations

```python
from specklepy.api import operations
from specklepy.transports.server import ServerTransport
from specklepy.transports.memory import MemoryTransport

# Send
transport = ServerTransport(client=client, stream_id="stream_id")
obj_hash = operations.send(base=base_obj, transports=[transport])

# Receive
received = operations.receive(obj_id=obj_hash, remote_transport=transport)
```

| Function | Signature | Returns |
|----------|-----------|---------|
| Send | `operations.send(base: Base, transports: List[AbstractTransport], use_default_cache: bool = True)` | `str` (object hash) |
| Receive | `operations.receive(obj_id: str, remote_transport: AbstractTransport, local_transport: AbstractTransport = None)` | `Base` |

### Version Creation (v3 API)

```python
from specklepy.core.api.inputs.version_inputs import CreateVersionInput

version_input = CreateVersionInput(
    project_id=project_id,
    model_id=model_id,
    object_id=object_id,
    message="Description of changes"
)
version = client.version.create(version_input)
```

### StreamWrapper

```python
from specklepy.api.wrapper import StreamWrapper

wrapper = StreamWrapper("https://app.speckle.systems/streams/abc123/commits/def456")
client = wrapper.get_client()
transport = wrapper.get_transport()
```

| Property | Type | Description |
|----------|------|-------------|
| `wrapper.stream_id` | str | Extracted stream ID |
| `wrapper.commit_id` | str | Extracted commit ID (if present) |
| `wrapper.branch_name` | str | Extracted branch name (if present) |
| `wrapper.host` | str | Server host URL |

---

## Speckle Object Types

### Base

```python
from specklepy.objects import Base

obj = Base()
obj.name = "MyObject"
obj["custom_property"] = "value"
obj["@detached_ref"] = other_base  # @ prefix = stored as separate reference
```

### Mesh

```python
from specklepy.objects.geometry import Mesh

mesh = Mesh()
mesh.vertices = [x0, y0, z0, x1, y1, z1, ...]  # flat list of floats
mesh.faces = [n, i0, i1, ..., n, i0, i1, ...]   # n = vertex count per face
mesh.colors = [argb0, argb1, ...]                 # optional vertex colors
mesh.units = "m"
```

**Face encoding**: Each face is preceded by its vertex count. Triangle: `[3, 0, 1, 2]`. Quad: `[4, 0, 1, 2, 3]`. N-gon: `[n, i0, i1, ..., in-1]`.

### Point

```python
from specklepy.objects.geometry import Point

pt = Point(x=1.0, y=2.0, z=3.0, units="m")
```

### RenderMaterial

```python
from specklepy.objects.other import RenderMaterial

mat = RenderMaterial()
mat.name = "Concrete"
mat.diffuse = 0xFFA0A0A0   # ARGB integer
mat.opacity = 1.0           # 0.0 = transparent, 1.0 = opaque
mat.metalness = 0.0         # 0.0 = dielectric, 1.0 = metallic
mat.roughness = 0.8         # 0.0 = mirror, 1.0 = rough
```

### Collection

```python
from specklepy.objects.other import Collection

coll = Collection(name="Level 1", collectionType="level")
coll.elements = [obj1, obj2, child_collection]
```

---

## Blender API Methods (Relevant to Speckle Integration)

### Mesh Data Access

```python
import bpy

obj = bpy.data.objects["MyObject"]
mesh = obj.data

# Vertices
for v in mesh.vertices:
    print(v.co.x, v.co.y, v.co.z)

# Triangulated faces (for Speckle export)
mesh.calc_loop_triangles()
for tri in mesh.loop_triangles:
    print(tri.vertices)  # tuple of 3 vertex indices

# Evaluated mesh (with modifiers applied)
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()
# ... extract data ...
obj_eval.to_mesh_clear()  # ALWAYS clean up
```

### Creating Mesh from Speckle Data

```python
mesh = bpy.data.meshes.new("ImportedMesh")
mesh.from_pydata(vertices_list, [], faces_list)
mesh.update()  # ALWAYS call after from_pydata

obj = bpy.data.objects.new("ImportedObject", mesh)
bpy.context.collection.objects.link(obj)  # REQUIRED to make visible
```

### Material Creation

```python
mat = bpy.data.materials.new("SpeckleMaterial")
mat.use_nodes = True
bsdf = mat.node_tree.nodes.get("Principled BSDF")

if bsdf:
    bsdf.inputs["Base Color"].default_value = (0.63, 0.63, 0.63, 1.0)
    bsdf.inputs["Metallic"].default_value = 0.0
    bsdf.inputs["Roughness"].default_value = 0.8
    bsdf.inputs["Alpha"].default_value = 1.0

# Assign to object
obj.data.materials.append(mat)
```

### Collection Management

```python
# Create collection
coll = bpy.data.collections.new("Speckle Import")
bpy.context.scene.collection.children.link(coll)

# Link object to collection
coll.objects.link(obj)
```

### Unit Settings

```python
scene = bpy.context.scene
scene.unit_settings.system = 'METRIC'
scene.unit_settings.length_unit = 'METERS'  # or 'MILLIMETERS', 'CENTIMETERS'
scene.unit_settings.scale_length = 1.0
```
