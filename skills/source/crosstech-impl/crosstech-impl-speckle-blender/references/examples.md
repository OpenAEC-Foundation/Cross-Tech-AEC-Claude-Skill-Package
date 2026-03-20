# crosstech-impl-speckle-blender — Examples

## Example 1: Send Selected Blender Objects to Speckle via SpecklePy

Programmatically sends all selected Blender objects as Speckle meshes.

```python
import bpy
from specklepy.objects import Base
from specklepy.objects.geometry import Mesh as SpeckleMesh
from specklepy.objects.other import Collection, RenderMaterial
from specklepy.transports.server import ServerTransport
from specklepy.api import operations
from specklepy.api.client import SpeckleClient
from specklepy.api.credentials import get_default_account

# Authenticate
client = SpeckleClient(host="https://app.speckle.systems")
account = get_default_account()
client.authenticate_with_account(account)

STREAM_ID = "your_stream_id"
transport = ServerTransport(client=client, stream_id=STREAM_ID)

def blender_mesh_to_speckle(bl_obj):
    """Convert a Blender mesh object to a Speckle Mesh."""
    # ALWAYS use evaluated mesh to include modifier results
    depsgraph = bpy.context.evaluated_depsgraph_get()
    obj_eval = bl_obj.evaluated_get(depsgraph)
    mesh_eval = obj_eval.to_mesh()
    mesh_eval.calc_loop_triangles()

    vertices = []
    for v in mesh_eval.vertices:
        vertices.extend([v.co.x, v.co.y, v.co.z])

    faces = []
    for tri in mesh_eval.loop_triangles:
        faces.append(3)
        faces.extend(tri.vertices)

    spk_mesh = SpeckleMesh()
    spk_mesh.vertices = vertices
    spk_mesh.faces = faces
    spk_mesh.units = "m"
    spk_mesh.name = bl_obj.name

    # Extract material if present
    if bl_obj.data.materials:
        bl_mat = bl_obj.data.materials[0]
        if bl_mat and bl_mat.use_nodes:
            bsdf = bl_mat.node_tree.nodes.get("Principled BSDF")
            if bsdf:
                color = bsdf.inputs["Base Color"].default_value
                render_mat = RenderMaterial()
                render_mat.name = bl_mat.name
                r, g, b = int(color[0]*255), int(color[1]*255), int(color[2]*255)
                render_mat.diffuse = (0xFF << 24) | (r << 16) | (g << 8) | b
                render_mat.opacity = bsdf.inputs["Alpha"].default_value
                render_mat.metalness = bsdf.inputs["Metallic"].default_value
                render_mat.roughness = bsdf.inputs["Roughness"].default_value
                spk_mesh["renderMaterial"] = render_mat

    obj_eval.to_mesh_clear()  # ALWAYS clean up evaluated mesh
    return spk_mesh

# Build root collection
root = Collection(name="Blender Export", collectionType="model")
root.elements = []

for obj in bpy.context.selected_objects:
    if obj.type == 'MESH':
        spk_mesh = blender_mesh_to_speckle(obj)
        root.elements.append(spk_mesh)

# Send and create version
obj_hash = operations.send(base=root, transports=[transport])
commit_id = client.commit.create(
    stream_id=STREAM_ID,
    object_id=obj_hash,
    message=f"Exported {len(root.elements)} objects from Blender"
)
print(f"Sent successfully. Commit: {commit_id}")
```

---

## Example 2: Receive Speckle Model into Blender with Materials

Downloads a Speckle model and recreates geometry with materials in Blender.

```python
import bpy
from specklepy.transports.server import ServerTransport
from specklepy.api import operations
from specklepy.api.client import SpeckleClient
from specklepy.api.credentials import get_default_account

client = SpeckleClient(host="https://app.speckle.systems")
account = get_default_account()
client.authenticate_with_account(account)

STREAM_ID = "your_stream_id"
transport = ServerTransport(client=client, stream_id=STREAM_ID)

# Get latest commit
commits = client.commit.list(STREAM_ID, limit=1)
latest = commits[0]

# Receive
received = operations.receive(
    obj_id=latest.referencedObject,
    remote_transport=transport
)

def argb_to_rgba(argb_int):
    """Convert Speckle ARGB integer to Blender RGBA tuple (0-1 range)."""
    a = ((argb_int >> 24) & 0xFF) / 255.0
    r = ((argb_int >> 16) & 0xFF) / 255.0
    g = ((argb_int >> 8) & 0xFF) / 255.0
    b = (argb_int & 0xFF) / 255.0
    return (r, g, b, a)

def create_blender_material(render_mat):
    """Create a Blender Principled BSDF material from a RenderMaterial."""
    mat = bpy.data.materials.new(render_mat.name or "SpeckleMaterial")
    mat.use_nodes = True
    bsdf = mat.node_tree.nodes.get("Principled BSDF")
    if bsdf:
        rgba = argb_to_rgba(render_mat.diffuse)
        bsdf.inputs["Base Color"].default_value = rgba
        bsdf.inputs["Metallic"].default_value = getattr(render_mat, 'metalness', 0.0)
        bsdf.inputs["Roughness"].default_value = getattr(render_mat, 'roughness', 0.5)
        bsdf.inputs["Alpha"].default_value = getattr(render_mat, 'opacity', 1.0)
    if getattr(render_mat, 'opacity', 1.0) < 1.0:
        mat.blend_method = 'BLEND'  # Blender 3.x EEVEE transparency
    return mat

def speckle_mesh_to_blender(spk_obj, parent_collection):
    """Convert a Speckle object with displayValue meshes to Blender objects."""
    display_meshes = getattr(spk_obj, 'displayValue', None) or []
    if not isinstance(display_meshes, list):
        display_meshes = [display_meshes]

    for i, spk_mesh in enumerate(display_meshes):
        if not hasattr(spk_mesh, 'vertices') or not hasattr(spk_mesh, 'faces'):
            continue

        # Parse vertices
        verts = []
        raw_verts = spk_mesh.vertices
        for j in range(0, len(raw_verts), 3):
            verts.append((raw_verts[j], raw_verts[j+1], raw_verts[j+2]))

        # Parse faces
        faces = []
        idx = 0
        raw_faces = spk_mesh.faces
        while idx < len(raw_faces):
            n = raw_faces[idx]
            face = tuple(raw_faces[idx+1 : idx+1+n])
            faces.append(face)
            idx += n + 1

        name = getattr(spk_obj, 'name', f"SpeckleObject_{i}")
        mesh = bpy.data.meshes.new(name)
        mesh.from_pydata(verts, [], faces)
        mesh.update()

        obj = bpy.data.objects.new(name, mesh)
        parent_collection.objects.link(obj)

        # Apply material if present
        render_mat = getattr(spk_obj, 'renderMaterial', None)
        if render_mat is None:
            render_mat = spk_obj.get("renderMaterial", None) if hasattr(spk_obj, 'get') else None
        if render_mat:
            bl_mat = create_blender_material(render_mat)
            obj.data.materials.append(bl_mat)

def process_collection(spk_coll, parent_bl_coll):
    """Recursively process Speckle Collections into Blender collections."""
    name = getattr(spk_coll, 'name', "Unnamed")
    bl_coll = bpy.data.collections.new(name)
    parent_bl_coll.children.link(bl_coll)

    elements = getattr(spk_coll, 'elements', []) or []
    for elem in elements:
        speckle_type = getattr(elem, 'speckle_type', '')
        if 'Collection' in str(speckle_type):
            process_collection(elem, bl_coll)
        else:
            speckle_mesh_to_blender(elem, bl_coll)

# Create root collection
root_coll = bpy.data.collections.new("Speckle Import")
bpy.context.scene.collection.children.link(root_coll)

# Process received data
if hasattr(received, 'elements'):
    process_collection(received, bpy.context.scene.collection)
else:
    speckle_mesh_to_blender(received, root_coll)

print("Receive complete.")
```

---

## Example 3: Send Blender Scene with Collection Hierarchy

Preserves the full Blender collection tree in Speckle.

```python
import bpy
from specklepy.objects.other import Collection
from specklepy.objects.geometry import Mesh as SpeckleMesh
from specklepy.transports.server import ServerTransport
from specklepy.api import operations

def collection_to_speckle(bl_coll):
    """Recursively convert a Blender collection hierarchy to Speckle."""
    spk_coll = Collection(name=bl_coll.name, collectionType="collection")
    spk_coll.elements = []

    # Convert child collections
    for child_coll in bl_coll.children:
        spk_coll.elements.append(collection_to_speckle(child_coll))

    # Convert objects in this collection
    for obj in bl_coll.objects:
        if obj.type == 'MESH':
            spk_mesh = blender_mesh_to_speckle(obj)  # from Example 1
            spk_coll.elements.append(spk_mesh)

    return spk_coll

# Convert the entire scene collection
root = collection_to_speckle(bpy.context.scene.collection)

transport = ServerTransport(client=client, stream_id=STREAM_ID)
obj_hash = operations.send(base=root, transports=[transport])
```

---

## Example 4: Unit-Aware Receive

ALWAYS verify and set units before creating geometry.

```python
def set_blender_units(speckle_units):
    """Set Blender scene units to match Speckle data."""
    unit_map = {
        "mm": ("METRIC", "MILLIMETERS"),
        "cm": ("METRIC", "CENTIMETERS"),
        "m": ("METRIC", "METERS"),
        "ft": ("IMPERIAL", "FEET"),
        "in": ("IMPERIAL", "INCHES"),
    }
    system, length = unit_map.get(speckle_units, ("METRIC", "METERS"))
    bpy.context.scene.unit_settings.system = system
    bpy.context.scene.unit_settings.length_unit = length

# Before receiving, check units on the root object
received = operations.receive(obj_id=obj_hash, remote_transport=transport)
speckle_units = getattr(received, 'units', 'm')
set_blender_units(speckle_units)

# Then proceed with mesh creation...
```

---

## Example 5: Testing with MemoryTransport (No Server Required)

```python
from specklepy.objects import Base
from specklepy.objects.geometry import Mesh as SpeckleMesh
from specklepy.transports.memory import MemoryTransport
from specklepy.api import operations

transport = MemoryTransport()

# Create test mesh
mesh = SpeckleMesh()
mesh.vertices = [0,0,0, 1,0,0, 1,1,0, 0,1,0]
mesh.faces = [4, 0, 1, 2, 3]
mesh.units = "m"

# Send to memory
obj_hash = operations.send(base=mesh, transports=[transport])

# Receive back
received = operations.receive(obj_id=obj_hash, remote_transport=transport)
assert len(received.vertices) == 12
assert received.units == "m"
```
