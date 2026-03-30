---
name: crosstech-impl-speckle-blender
description: >
  Use when sending or receiving BIM data between Speckle and Blender via the
  Speckle connector or SpecklePy. Prevents the common mistake of expecting
  full IFC fidelity through Speckle or losing materials during conversion.
  Covers Speckle Base to Blender mesh/curve conversion, material mapping,
  collection structure, Bonsai/IFC data through Speckle, and SpecklePy API.
  Keywords: Speckle, Blender, SpecklePy, connector, BIM data, mesh conversion,
  material mapping, Bonsai, IFC through Speckle, send receive, BIM to Blender,
  render BIM model, Blender visualization, share model.
license: MIT
compatibility: "Designed for Claude Code. Covers Speckle 2.x/3.x, Blender 3.x/4.x, SpecklePy 2.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-impl-speckle-blender

## Quick Reference

### Technology Boundary

| Aspect | Side A: Speckle | Side B: Blender |
|--------|----------------|-----------------|
| Data model | Base object graph (hash-based, immutable) | bpy.data collections (mutable, ID blocks) |
| Geometry | Mesh, Curve, Polycurve, Point, NURBS | Mesh, Bezier Curve, NURBS Curve, Empty |
| Materials | RenderMaterial (diffuse, opacity, metalness, roughness) | Principled BSDF node tree (full PBR) |
| Hierarchy | Collection with `elements` list | bpy.data.collections with linked objects |
| Units | String property on Base (`mm`, `m`, `cm`) | Scene unit system (`bpy.context.scene.unit_settings`) |
| Storage | Server, SQLite cache, Memory transport | .blend file (binary) |
| Versions | Speckle Server 2.x / 3.x, SpecklePy 2.x | Blender 3.x / 4.x |

### The Bridge: Speckle Blender Connector

The Speckle Blender connector is a Blender add-on that converts between Speckle Base objects and Blender data blocks. It uses SpecklePy internally for transport and serialization.

| Direction | What Happens |
|-----------|-------------|
| Blender -> Speckle (Send) | Mesh/Curve data extracted from bpy.data, materials reduced to RenderMaterial, collections mapped to Speckle Collections |
| Speckle -> Blender (Receive) | Base objects converted to bpy.data.meshes/curves, RenderMaterials recreated as Principled BSDF, Collections mapped to bpy.data.collections |

### Supported Conversions

| Blender Type | Speckle Type | Direction |
|-------------|-------------|-----------|
| Mesh | Mesh | Bidirectional |
| Bezier Curve | Curve / Polycurve | Bidirectional |
| NURBS Curve | Curve / Polycurve | Bidirectional |
| Empty / Collection | Collection | Bidirectional |
| Material (Principled BSDF) | RenderMaterial | Bidirectional |

### Critical Warnings

**NEVER** expect full IFC fidelity through Speckle -- Speckle transmits geometry and basic properties, NOT parametric definitions or IFC relationships.

**NEVER** assume complex Blender shader node trees survive the round-trip -- only the first Principled BSDF node's basic parameters (diffuse color, opacity, metalness, roughness) are extracted.

**NEVER** rely on Bonsai/IFC metadata surviving a Speckle round-trip -- Bonsai's native IFC object model is richer than Speckle's Base representation.

**ALWAYS** verify unit settings match between Speckle and Blender before send/receive -- unit mismatch causes geometry scaling errors.

**ALWAYS** use `applicationId` tracking when updating previously received models -- this prevents duplicate objects on subsequent receives.

---

## Side A: Speckle Data Model

### Base Object Fundamentals

Every object in Speckle inherits from `Base`. Key properties:

| Property | Type | Purpose |
|----------|------|---------|
| `id` | string | Hash-based unique ID (content-addressable) |
| `applicationId` | string | Host application identity (enables update tracking) |
| `speckle_type` | string | Type discriminator |
| `units` | string | Unit system for geometry values |
| `displayValue` | List[Mesh] | Visual representation meshes |

### Speckle Collections

Collections represent hierarchical groupings:

```python
from specklepy.objects import Base
from specklepy.objects.other import Collection

root = Collection(name="Building", collectionType="model")
level1 = Collection(name="Level 1", collectionType="level")
root.elements = [level1]
level1.elements = [wall_obj, door_obj]
```

Collections MUST form a directed tree -- no cycles allowed.

### RenderMaterial

```python
from specklepy.objects.other import RenderMaterial

mat = RenderMaterial()
mat.name = "Concrete"
mat.diffuse = 0xFFA0A0A0   # ARGB integer
mat.opacity = 1.0
mat.metalness = 0.0
mat.roughness = 0.8
```

**CRITICAL**: RenderMaterial supports ONLY diffuse color, opacity, metalness, and roughness. Texture maps, normal maps, emission, subsurface scattering, and all other PBR channels are NOT supported.

### Proxy Collections (v3)

In Speckle v3, organizational data is stored at the root level as proxy collections:

- `renderMaterialProxies` -- material assignments
- `levelProxies` -- building levels
- `instanceDefinitionProxies` -- block/component definitions

Each proxy references objects by `applicationId`, requiring a two-step lookup.

---

## Side B: Blender Data Model

### Relevant bpy.data Collections

| Collection | Type | Maps To |
|-----------|------|---------|
| `bpy.data.meshes` | Mesh geometry | Speckle Mesh |
| `bpy.data.curves` | Bezier/NURBS curves | Speckle Curve/Polycurve |
| `bpy.data.objects` | Object wrappers | Speckle Base with displayValue |
| `bpy.data.collections` | Hierarchy groups | Speckle Collection |
| `bpy.data.materials` | Shader node trees | Speckle RenderMaterial |

### Blender Material Structure

A Blender Principled BSDF material exposes ~20+ parameters. The Speckle connector extracts ONLY:

| Principled BSDF Input | RenderMaterial Property | Preserved? |
|----------------------|------------------------|------------|
| Base Color | `diffuse` | YES |
| Alpha | `opacity` | YES |
| Metallic | `metalness` | YES |
| Roughness | `roughness` | YES |
| Normal Map | -- | NO |
| Emission | -- | NO |
| Subsurface | -- | NO |
| Specular | -- | NO |
| All texture nodes | -- | NO |

---

## The Bridge: Send/Receive Workflows

### Workflow 1: Send from Blender to Speckle (Connector UI)

1. Open the Speckle panel in Blender sidebar
2. Authenticate with your Speckle account
3. Select the project and model (branch)
4. Choose objects to send (selection, collection, or all)
5. Click Send -- the connector converts Blender data to Base objects and pushes to the server

### Workflow 2: Receive from Speckle into Blender (Connector UI)

1. Open the Speckle panel in Blender sidebar
2. Select the project, model, and version to receive
3. Click Receive -- the connector downloads Base objects and converts to Blender data
4. Objects appear in a new collection named after the model

### Workflow 3: SpecklePy Programmatic Send

```python
import bpy
from specklepy.objects import Base
from specklepy.objects.geometry import Mesh as SpeckleMesh
from specklepy.transports.server import ServerTransport
from specklepy.api import operations
from specklepy.api.client import SpeckleClient
from specklepy.api.credentials import get_default_account

# Authenticate
client = SpeckleClient(host="https://app.speckle.systems")
account = get_default_account()
client.authenticate_with_account(account)

# Extract mesh data from Blender
bl_obj = bpy.data.objects["MyObject"]
bl_mesh = bl_obj.data
bl_mesh.calc_loop_triangles()

vertices = []
for v in bl_mesh.vertices:
    vertices.extend([v.co.x, v.co.y, v.co.z])

faces = []
for tri in bl_mesh.loop_triangles:
    faces.append(3)  # triangle face count
    faces.extend(tri.vertices)

# Create Speckle Mesh
spk_mesh = SpeckleMesh()
spk_mesh.vertices = vertices
spk_mesh.faces = faces
spk_mesh.units = "m"

# Send
transport = ServerTransport(client=client, stream_id="your_stream_id")
obj_hash = operations.send(base=spk_mesh, transports=[transport])

# Create version
commit_id = client.commit.create(
    stream_id="your_stream_id",
    object_id=obj_hash,
    message="Blender mesh export"
)
```

### Workflow 4: SpecklePy Programmatic Receive

```python
from specklepy.transports.server import ServerTransport
from specklepy.api import operations

transport = ServerTransport(client=client, stream_id="your_stream_id")

# Get latest commit
commits = client.commit.list("your_stream_id", limit=1)
latest = commits[0]

# Receive object graph
received = operations.receive(
    obj_id=latest.referencedObject,
    remote_transport=transport
)

# Convert to Blender mesh
def speckle_mesh_to_blender(spk_mesh, name="SpeckleImport"):
    verts = []
    for i in range(0, len(spk_mesh.vertices), 3):
        verts.append((
            spk_mesh.vertices[i],
            spk_mesh.vertices[i + 1],
            spk_mesh.vertices[i + 2]
        ))

    faces = []
    idx = 0
    while idx < len(spk_mesh.faces):
        n = spk_mesh.faces[idx]
        face = tuple(spk_mesh.faces[idx + 1 : idx + 1 + n])
        faces.append(face)
        idx += n + 1

    mesh = bpy.data.meshes.new(name)
    mesh.from_pydata(verts, [], faces)
    mesh.update()

    obj = bpy.data.objects.new(name, mesh)
    bpy.context.collection.objects.link(obj)
    return obj
```

---

## IFC Data Through Speckle

### One-Way Limitation: Bonsai -> Speckle (Send Only)

The IFC/Bonsai to Speckle workflow is ONE-DIRECTIONAL:

| Direction | Status | Result |
|-----------|--------|--------|
| Bonsai -> Speckle (Send) | WORKS | IFC elements sent as Speckle Base objects with geometry + basic properties |
| Speckle -> Bonsai (Receive) | DOES NOT WORK | Received objects are plain Blender meshes, NOT recognized as IFC elements by Bonsai |

### What Transfers (IFC -> Speckle -> Blender)

| IFC Data | Transferred? | Notes |
|----------|-------------|-------|
| Mesh geometry | YES | Triangulated during conversion |
| Material assignments | YES | Basic properties only (diffuse, opacity) |
| Spatial hierarchy | YES | Mapped to Speckle Collections -> Blender collections |
| Basic property sets | PARTIAL | Flattened to dynamic properties on Base |
| IfcRelAggregates | NO | Relationship semantics lost |
| IfcRelContainedInSpatialStructure | NO | Spatial containment details lost |
| Parametric definitions | NO | Only display mesh transferred |
| Bonsai editing capability | NO | Cannot edit as IFC after round-trip |

### Data Loss Decision Tree

```
Is full IFC fidelity required?
├── YES → Do NOT use Speckle as transport. Use native IFC files directly.
└── NO → Is visual review sufficient?
    ├── YES → Speckle is appropriate. Geometry + basic materials transfer.
    └── NO → Do you need property data?
        ├── YES (basic props) → Speckle works. Properties flatten but transfer.
        └── YES (full psets) → Use IFC files directly or IfcOpenShell.
```

---

## Collection Structure Mapping

### Blender -> Speckle

```
Blender Scene Collection           Speckle Root Collection
├── Building                  →    ├── Collection("Building")
│   ├── Level 1               →    │   ├── Collection("Level 1")
│   │   ├── Wall.001          →    │   │   ├── Base(displayValue=[Mesh])
│   │   └── Door.001          →    │   │   └── Base(displayValue=[Mesh])
│   └── Level 2               →    │   └── Collection("Level 2")
└── Site                      →    └── Collection("Site")
```

### Speckle -> Blender

On receive, the connector creates a root collection named `[ProjectName] - [ModelName]`. All received objects are placed inside, preserving the source hierarchy.

---

## Round-Trip Data Integrity

### What SURVIVES Blender -> Speckle -> Blender

- Mesh vertex positions and face topology
- Basic material properties (diffuse, opacity, metalness, roughness)
- Collection hierarchy and object names
- Object transforms (location, rotation, scale)

### What is LOST or DEGRADED

- Complex shader node trees (reduced to single Principled BSDF parameters)
- Texture maps and UV-linked images
- Modifiers (ALWAYS apply modifiers before sending)
- Shape keys / blend shapes
- Particle systems
- Physics simulations
- Constraints and drivers
- Custom properties on objects (use Speckle dynamic properties instead)
- Vertex groups (partial support)

**ALWAYS** apply all modifiers before sending if the modified geometry is needed. The connector reads the evaluated mesh, but modifier stacks are NOT preserved.

---

## Unit Handling

| Blender Unit | Speckle Unit String | Notes |
|-------------|-------------------|-------|
| Meters | `"m"` | Default for architectural scenes |
| Millimeters | `"mm"` | Common for BIM data from Revit |
| Centimeters | `"cm"` | |
| None | `"m"` | Blender defaults to meters when unit system is "None" |

**ALWAYS** set `bpy.context.scene.unit_settings.length_unit` to match the expected Speckle data units before receiving. Unit mismatch causes geometry to appear at wrong scale.

---

## Reference Links

- [references/methods.md](references/methods.md) -- SpecklePy API methods and Blender connector operations
- [references/examples.md](references/examples.md) -- Working code examples for common workflows
- [references/anti-patterns.md](references/anti-patterns.md) -- What NOT to do at the Speckle-Blender boundary

### Official Sources

- https://speckle.guide/user/blender.html
- https://speckle.systems/docs/developer/python
- https://specklepy.readthedocs.io/
- https://docs.blender.org/api/current/
- https://github.com/specklesystems/speckle-blender
