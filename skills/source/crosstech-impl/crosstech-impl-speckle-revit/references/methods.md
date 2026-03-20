# crosstech-impl-speckle-revit — Methods Reference

## Speckle .NET SDK Core Operations

### Operations.Send

Serializes a Base object and writes it to one or more transports.

```csharp
// Signature
public static async Task<string> Send(
    Base @object,
    IReadOnlyList<ITransport> transports,
    bool useDefaultCache = true,
    Action<ConcurrentDictionary<string, int>>? onProgressAction = null,
    CancellationToken cancellationToken = default
);

// Returns: object hash (string)
```

**Parameters:**
- `@object` — The root Base object to send
- `transports` — One or more transport destinations (ServerTransport, SQLiteTransport, MemoryTransport)
- `useDefaultCache` — When true, uses the global SQLite cache alongside specified transports
- `onProgressAction` — Callback reporting serialization progress per transport

### Operations.Receive

Deserializes a Base object from a remote transport.

```csharp
// Signature
public static async Task<Base> Receive(
    string objectId,
    ITransport? remoteTransport = null,
    ITransport? localTransport = null,
    Action<ConcurrentDictionary<string, int>>? onProgressAction = null,
    CancellationToken cancellationToken = default
);

// Returns: deserialized Base object
```

**Parameters:**
- `objectId` — The hash of the root object to receive
- `remoteTransport` — Source transport (typically ServerTransport)
- `localTransport` — Optional local cache transport

### Client.CommitCreate

Creates a new version (commit) referencing a root object.

```csharp
public async Task<string> CommitCreate(CommitCreateInput input);

// CommitCreateInput properties:
// - streamId: string (required)
// - objectId: string (required, from Operations.Send)
// - branchName: string (default "main")
// - message: string (commit message)
// - sourceApplication: string (e.g., "Revit")
// - parents: List<string> (previous commit IDs)
```

### ServerTransport Constructor

```csharp
public ServerTransport(Account account, string streamId, int timeoutSeconds = 60);
```

### SQLiteTransport Constructor

```csharp
public SQLiteTransport(string? basePath = null, string? applicationName = null);
```

---

## Speckle Revit Connector Internal Methods

### RevitRootObjectBuilder

Converts Revit elements to Speckle Base objects. NOT directly callable — used internally by the connector.

**Key responsibilities:**
- Reads `Element.Parameters` collection via Revit API
- Extracts geometry via `Element.get_Geometry(Options)`
- Builds `displayValue` meshes at configured Detail Level
- Sets `applicationId` from `Element.UniqueId`

### RevitToSpeckleCacheSingleton

Caches converted Speckle objects to avoid redundant processing.

**Behavior:**
- Key: Revit `ElementId` + geometry hash
- Value: Serialized Speckle Base object
- Cache is cleared when the document changes or connector restarts

### RevitMaterialBaker

Maps Speckle RenderMaterial to Revit Material.

**Mapping rules:**
| Speckle Property | Revit Property |
|-----------------|---------------|
| `diffuse` (ARGB int) | `Material.Color` |
| `opacity` (0.0-1.0) | `Material.Transparency` (0-100, inverted) |
| `metalness` (0.0-1.0) | No direct equivalent; approximated via appearance asset |
| `roughness` (0.0-1.0) | No direct equivalent; approximated via appearance asset |

### RevitGroupBaker

Creates Revit Groups from Speckle Collections.

**Behavior:**
- Speckle Collection with `collectionType` = "Group" creates a Revit `Group`
- Nested collections create nested groups (Revit 2022+ only)
- Groups containing only DirectShapes receive the group name as a parameter

---

## Revit API Methods Used by the Connector

### Element.get_Geometry

```csharp
// Get element geometry for conversion
Options geomOptions = new Options
{
    DetailLevel = ViewDetailLevel.Fine,  // Controls tessellation
    ComputeReferences = true,
    IncludeNonVisibleObjects = false
};
GeometryElement geom = element.get_Geometry(geomOptions);
```

### DirectShape.CreateElement

```csharp
// Create a DirectShape from Speckle geometry
DirectShape ds = DirectShape.CreateElement(
    doc,                              // Revit Document
    new ElementId(BuiltInCategory.OST_GenericModel)  // Category
);
ds.SetShape(solidList);               // List<GeometryObject>
ds.SetName("Speckle-Wall-abc123");    // Display name
```

### Transaction Pattern

```csharp
// ALL Revit modifications MUST be inside a Transaction
using (Transaction t = new Transaction(doc, "Speckle Receive"))
{
    t.Start();
    // Create elements, set parameters, apply materials
    TransactionStatus status = t.Commit();
    // status == Committed on success, RolledBack on failure
}
```

### Parameter Setting

```csharp
// Set parameter value on a Revit element
Parameter param = element.LookupParameter("Mark");
if (param != null && !param.IsReadOnly)
{
    param.Set("Speckle-Import-001");
}

// Set numeric parameter with unit conversion
Parameter offset = element.LookupParameter("Base Offset");
if (offset != null)
{
    // Revit internal units are feet; convert from mm
    offset.Set(value_mm / 304.8);
}
```

---

## SpecklePy Methods (Python SDK — for automation scripts)

### operations.send

```python
from specklepy.api import operations
from specklepy.transports.server import ServerTransport

transport = ServerTransport(client=client, stream_id="stream-id")
object_hash = operations.send(base=my_base_object, transports=[transport])
```

### operations.receive

```python
received = operations.receive(
    obj_id="object-hash",
    remote_transport=transport
)
```

### StreamWrapper

```python
from specklepy.api.wrapper import StreamWrapper

wrapper = StreamWrapper("https://app.speckle.systems/streams/abc/commits/def")
client = wrapper.get_client()
transport = wrapper.get_transport()
```

### Base Object Dynamic Properties

```python
from specklepy.objects import Base

obj = Base()
obj.name = "Wall-001"
obj["custom_param"] = 42
obj["@detached_geometry"] = mesh_object  # @ prefix = detached storage
```

---

## Type Mapping Methods

### Automatic Type Matching

The connector resolves Revit types using a three-level hierarchy:

1. **Category** — `BuiltInCategory` enum (e.g., `OST_Walls`, `OST_Doors`)
2. **Family** — Family name string (e.g., `Basic Wall`, `M_Single-Flush`)
3. **Type** — Type name string (e.g., `Generic - 200mm`, `0915 x 2134mm`)

```
Speckle object → properties.category → match BuiltInCategory
               → properties.family   → match Family name
               → properties.type     → match FamilySymbol name
```

If ALL three match, the element is created as a native Revit family instance.
If ANY level fails to match, the element falls back to DirectShape.

### Manual Type Mapping

Exposed via the connector UI. The user maps:
- Source Speckle category → Target Revit category
- Source family name → Target Revit family
- Source type name → Target Revit family type

Manual mappings are saved per project and persist across sessions.
