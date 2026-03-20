# crosstech-impl-speckle-blender — Anti-Patterns

## AP-01: Expecting Full IFC Fidelity Through Speckle

**WRONG**: Treating Speckle as an IFC transport mechanism.

```python
# WRONG — expecting IFC relationships to survive Speckle transport
received = operations.receive(obj_id=commit.referencedObject, remote_transport=transport)
# Attempting to access IfcRelAggregates or IfcRelContainedInSpatialStructure
relationships = received.get("IfcRelAggregates")  # ALWAYS returns None
```

**WHY**: Speckle converts IFC elements to Base objects with flattened properties. IFC relationships (IfcRelAggregates, IfcRelContainedInSpatialStructure, IfcRelDefinesByProperties) are NOT preserved as structured data. Parametric definitions are replaced by display meshes.

**CORRECT**: Use native IFC files with IfcOpenShell when full IFC fidelity is required. Use Speckle only when visual geometry and basic properties are sufficient.

---

## AP-02: Assuming Bonsai Recognizes Speckle-Received Data

**WRONG**: Receiving Speckle data and expecting Bonsai (BlenderBIM) to treat it as editable IFC elements.

```python
# WRONG — Bonsai cannot edit Speckle-received objects as IFC
# Step 1: Receive from Speckle into Blender
# Step 2: Open Bonsai panel, expect to see IFC properties
# Result: Objects appear as plain Blender meshes, NOT as IFC elements
```

**WHY**: The Speckle-to-Blender receive pipeline creates standard Blender mesh objects. Bonsai requires its own native IFC data model (stored via IfcOpenShell) that is fundamentally different from Speckle's Base object representation. The Bonsai -> Speckle direction works (send), but Speckle -> Bonsai does NOT.

**CORRECT**: For IFC editing workflows, open IFC files directly in Bonsai. Use Speckle only for visualization and coordination, not for IFC authoring round-trips.

---

## AP-03: Sending Complex Shader Node Trees via Speckle

**WRONG**: Building elaborate Blender material setups and expecting them to transfer.

```python
# WRONG — complex material setup that will be lost
mat = bpy.data.materials.new("DetailedConcrete")
mat.use_nodes = True
tree = mat.node_tree
# Adding noise texture, color ramp, normal map, bump node...
# None of these survive the Speckle round-trip
```

**WHY**: The Speckle connector extracts ONLY four properties from the first Principled BSDF node: diffuse color, opacity, metalness, and roughness. All texture nodes, normal maps, emission, subsurface scattering, and procedural setups are silently discarded during send.

**CORRECT**: Accept that material transfer is limited to basic properties. If complex materials are needed on the receiving end, create them after receive using the basic RenderMaterial as a starting point.

```python
# CORRECT — build material on receive side
def enhance_received_material(bl_mat):
    """Add complexity after receiving basic Speckle material."""
    bsdf = bl_mat.node_tree.nodes.get("Principled BSDF")
    # Add texture nodes, normal maps etc. AFTER receive
    # The base color from Speckle serves as a reference
```

---

## AP-04: Forgetting to Apply Modifiers Before Send

**WRONG**: Sending objects with unapplied modifiers expecting the modified geometry to transfer.

```python
# WRONG — modifier stack not explicitly applied
obj = bpy.data.objects["WallWithBevel"]
# obj has a Bevel modifier, but sending raw mesh data
mesh = obj.data  # This is the BASE mesh, not the modified mesh
```

**WHY**: The Speckle connector reads the evaluated mesh (with modifiers applied), but the modifier STACK is NOT preserved. On receive, you get the final mesh without any modifier history. If you manually extract mesh data via SpecklePy, you MUST use the evaluated depsgraph to get post-modifier geometry.

**CORRECT**: When using SpecklePy directly, ALWAYS use the evaluated mesh.

```python
# CORRECT — use evaluated mesh
depsgraph = bpy.context.evaluated_depsgraph_get()
obj_eval = obj.evaluated_get(depsgraph)
mesh_eval = obj_eval.to_mesh()
# Extract vertices and faces from mesh_eval
obj_eval.to_mesh_clear()  # ALWAYS clean up
```

---

## AP-05: Ignoring Unit Mismatch

**WRONG**: Receiving Speckle data without checking or setting Blender units.

```python
# WRONG — receiving millimeter data into a meter-scale scene
# Speckle model was created in Revit (millimeters)
# Blender scene is set to meters
# Result: model appears 1000x too large or objects are microscopic
received = operations.receive(obj_id=obj_hash, remote_transport=transport)
```

**WHY**: Speckle stores a `units` string on Base objects, but does NOT automatically scale geometry during conversion. If the Blender scene unit does not match the Speckle data unit, geometry appears at the wrong scale.

**CORRECT**: ALWAYS check and set Blender units before creating geometry from Speckle data.

```python
# CORRECT — match units before receive
speckle_units = getattr(received, 'units', 'm')
unit_map = {"mm": "MILLIMETERS", "cm": "CENTIMETERS", "m": "METERS"}
bpy.context.scene.unit_settings.system = 'METRIC'
bpy.context.scene.unit_settings.length_unit = unit_map.get(speckle_units, "METERS")
```

---

## AP-06: Creating Duplicate Objects on Re-Receive

**WRONG**: Receiving from the same Speckle source multiple times without handling existing objects.

```python
# WRONG — second receive creates duplicates
received_v1 = operations.receive(obj_id=hash_v1, remote_transport=transport)
# ... create Blender objects ...
received_v2 = operations.receive(obj_id=hash_v2, remote_transport=transport)
# ... create Blender objects again — now you have duplicates
```

**WHY**: SpecklePy's `operations.receive` returns data objects only -- it does NOT manage Blender scene state. Each receive + mesh creation cycle adds new objects. The Speckle Blender connector UI handles this via `applicationId` tracking, but programmatic workflows MUST implement deduplication manually.

**CORRECT**: Track `applicationId` values and update existing objects instead of creating new ones.

```python
# CORRECT — update existing objects by applicationId
app_id = getattr(spk_obj, 'applicationId', None)
if app_id:
    existing = next(
        (o for o in bpy.data.objects if o.get("speckle_app_id") == app_id),
        None
    )
    if existing:
        # Update existing object's mesh data
        update_mesh_data(existing, spk_obj)
    else:
        # Create new object and tag it
        new_obj = create_blender_object(spk_obj)
        new_obj["speckle_app_id"] = app_id
```

---

## AP-07: Running SpecklePy from a Background Thread in Blender

**WRONG**: Calling bpy API from a thread spawned for async Speckle operations.

```python
import threading

# WRONG — bpy calls from non-main thread
def async_receive():
    received = operations.receive(obj_id=obj_hash, remote_transport=transport)
    # This crashes or produces undefined behavior:
    mesh = bpy.data.meshes.new("ThreadedMesh")

threading.Thread(target=async_receive).start()
```

**WHY**: Blender's bpy API is NOT thread-safe. All bpy calls MUST execute on the main thread. SpecklePy's transport operations (network I/O) can run on background threads, but creating Blender objects MUST happen on the main thread.

**CORRECT**: Use SpecklePy for data download in background, then create Blender objects on the main thread via a timer or modal operator.

```python
# CORRECT — separate data download from Blender object creation
import threading

received_data = [None]

def download():
    received_data[0] = operations.receive(
        obj_id=obj_hash, remote_transport=transport
    )

thread = threading.Thread(target=download)
thread.start()
thread.join()  # Or use bpy.app.timers for non-blocking

# Main thread: create Blender objects
if received_data[0]:
    create_blender_objects(received_data[0])
```

---

## AP-08: Sending Empty or Invalid Mesh Data

**WRONG**: Creating Speckle Mesh objects with malformed vertex/face data.

```python
# WRONG — face indices reference non-existent vertices
mesh = SpeckleMesh()
mesh.vertices = [0,0,0, 1,0,0]  # Only 2 vertices (indices 0, 1)
mesh.faces = [3, 0, 1, 2]        # Face references vertex index 2 — does not exist
```

**WHY**: Speckle does not validate mesh topology. Invalid face indices cause crashes or invisible geometry on the receiving end. Blender's `from_pydata` will raise an exception or produce corrupted geometry.

**CORRECT**: ALWAYS validate that face indices are within the vertex count range before sending.

```python
# CORRECT — validate before send
vertex_count = len(mesh.vertices) // 3
idx = 0
while idx < len(mesh.faces):
    n = mesh.faces[idx]
    for j in range(1, n + 1):
        assert mesh.faces[idx + j] < vertex_count, f"Face index {mesh.faces[idx + j]} >= vertex count {vertex_count}"
    idx += n + 1
```
