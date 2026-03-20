# crosstech-impl-speckle-revit — Examples

## Example 1: Send Revit Walls to Speckle via .NET SDK

Use this pattern when automating Revit-to-Speckle exports from a Revit add-in or Dynamo script.

```csharp
using Autodesk.Revit.DB;
using Speckle.Core.Api;
using Speckle.Core.Credentials;
using Speckle.Core.Models;
using Speckle.Core.Transports;

public async Task<string> SendWallsToSpeckle(Document doc, string streamId)
{
    // Step 1: Authenticate
    var account = AccountManager.GetDefaultAccount();
    var client = new Client(account);
    var transport = new ServerTransport(account, streamId);

    // Step 2: Collect walls from the active document
    var collector = new FilteredElementCollector(doc)
        .OfClass(typeof(Wall))
        .WhereElementIsNotElementType();

    // Step 3: Build root Base object with wall data
    var root = new Base();
    root["category"] = "Walls";
    var wallList = new List<Base>();

    foreach (Wall wall in collector)
    {
        var speckleWall = new Base();
        speckleWall["applicationId"] = wall.UniqueId;
        speckleWall["family"] = wall.WallType.FamilyName;
        speckleWall["type"] = wall.WallType.Name;

        // Extract parameters
        var parameters = new Dictionary<string, object>();
        foreach (Parameter p in wall.Parameters)
        {
            if (p.HasValue)
            {
                parameters[p.Definition.Name] = GetParameterValue(p);
            }
        }
        speckleWall["parameters"] = parameters;

        wallList.Add(speckleWall);
    }

    root["@elements"] = wallList;  // @ = detached for performance

    // Step 4: Send to Speckle
    string objectId = await Operations.Send(root, new[] { transport });

    // Step 5: Create a version (commit)
    string commitId = await client.CommitCreate(new CommitCreateInput
    {
        streamId = streamId,
        objectId = objectId,
        branchName = "revit-walls",
        message = $"Exported {wallList.Count} walls from Revit",
        sourceApplication = "Revit"
    });

    return commitId;
}

private object GetParameterValue(Parameter p)
{
    switch (p.StorageType)
    {
        case StorageType.Double:
            return p.AsDouble();
        case StorageType.Integer:
            return p.AsInteger();
        case StorageType.String:
            return p.AsString() ?? "";
        case StorageType.ElementId:
            return p.AsElementId().IntegerValue;
        default:
            return null;
    }
}
```

---

## Example 2: Receive Speckle Objects as DirectShapes in Revit

Use this pattern when receiving non-Revit geometry (from Blender, Rhino, or custom scripts) into Revit.

```csharp
using Autodesk.Revit.DB;
using Speckle.Core.Api;
using Speckle.Core.Credentials;
using Speckle.Core.Models;
using Speckle.Core.Transports;

public async Task ReceiveAsDirectShapes(Document doc, string streamId, string commitId)
{
    // Step 1: Authenticate and receive
    var account = AccountManager.GetDefaultAccount();
    var client = new Client(account);
    var transport = new ServerTransport(account, streamId);

    var commit = await client.CommitGet(streamId, commitId);
    var rootObject = await Operations.Receive(
        commit.referencedObject,
        remoteTransport: transport
    );

    // Step 2: Create DirectShapes in a transaction
    using (Transaction t = new Transaction(doc, "Speckle Receive"))
    {
        t.Start();

        foreach (var child in rootObject.GetDynamicMembers())
        {
            var speckleObj = rootObject[child] as Base;
            if (speckleObj == null) continue;

            // Determine category (default to GenericModel)
            var categoryStr = speckleObj["category"]?.ToString() ?? "Generic Models";
            var categoryId = ResolveCategoryId(categoryStr);

            // Create DirectShape
            DirectShape ds = DirectShape.CreateElement(doc, categoryId);

            // Convert Speckle mesh to Revit solid
            // (simplified — real implementation uses TessellatedShapeBuilder)
            var meshData = speckleObj["displayValue"] as List<object>;
            if (meshData != null)
            {
                var solids = ConvertSpeckleMeshToRevitSolids(meshData);
                ds.SetShape(solids);
            }

            // Set name from applicationId
            var appId = speckleObj["applicationId"]?.ToString();
            if (appId != null)
            {
                ds.SetName($"Speckle-{appId}");
            }
        }

        t.Commit();
    }
}

private ElementId ResolveCategoryId(string category)
{
    // Map common Speckle categories to Revit BuiltInCategory
    var mapping = new Dictionary<string, BuiltInCategory>
    {
        { "Walls", BuiltInCategory.OST_Walls },
        { "Floors", BuiltInCategory.OST_Floors },
        { "Roofs", BuiltInCategory.OST_Roofs },
        { "Doors", BuiltInCategory.OST_Doors },
        { "Windows", BuiltInCategory.OST_Windows },
        { "Columns", BuiltInCategory.OST_Columns },
        { "Generic Models", BuiltInCategory.OST_GenericModel },
    };

    if (mapping.TryGetValue(category, out var bic))
        return new ElementId(bic);

    return new ElementId(BuiltInCategory.OST_GenericModel);
}
```

---

## Example 3: Python Automation — Read Revit Data from Speckle

Use this pattern when processing Revit-originated BIM data in a Python pipeline (CI/CD, analysis, reporting).

```python
from specklepy.api.client import SpeckleClient
from specklepy.api.credentials import get_default_account
from specklepy.transports.server import ServerTransport
from specklepy.api import operations

# Step 1: Connect
client = SpeckleClient(host="https://app.speckle.systems")
account = get_default_account()
client.authenticate_with_account(account)

# Step 2: Get latest version from the Revit model branch
stream_id = "your-stream-id"
branch = client.branch.get(stream_id, "revit-structural")
latest_commit = branch.commits.items[0]

# Step 3: Receive the data
transport = ServerTransport(client=client, stream_id=stream_id)
root = operations.receive(
    obj_id=latest_commit.referencedObject,
    remote_transport=transport
)

# Step 4: Traverse and extract wall data (Speckle v3 pattern)
def extract_elements(obj, category_filter="Walls"):
    """Recursively extract elements matching a category."""
    results = []

    # Check if this object has the target category
    props = getattr(obj, "properties", None) or {}
    if isinstance(props, dict) and props.get("category") == category_filter:
        results.append(obj)

    # Traverse children
    for attr_name in obj.get_dynamic_member_names():
        child = getattr(obj, attr_name, None)
        if isinstance(child, list):
            for item in child:
                if hasattr(item, "get_dynamic_member_names"):
                    results.extend(extract_elements(item, category_filter))
        elif hasattr(child, "get_dynamic_member_names"):
            results.extend(extract_elements(child, category_filter))

    return results

walls = extract_elements(root, "Walls")
print(f"Found {len(walls)} walls")

# Step 5: Extract parameters for reporting
for wall in walls:
    params = getattr(wall, "parameters", {}) or {}
    type_params = params.get("Type Parameters", {})
    instance_params = params.get("Instance Parameters", {})

    width = type_params.get("Width", {}).get("value", "N/A")
    fire_rating = type_params.get("Fire Rating", {}).get("value", "N/A")
    base_offset = instance_params.get("Base Offset", {}).get("value", "N/A")

    print(f"Wall: width={width}, fire_rating={fire_rating}, base_offset={base_offset}")
```

---

## Example 4: View-Based Filtering for Selective Send

Use the connector UI to send only visible elements from a specific Revit view. This is the recommended workflow for large models.

**Workflow steps:**

1. Create a dedicated Revit view (e.g., "Speckle Export - Structural")
2. Apply Visibility/Graphics overrides to show only target categories
3. Apply view filters to further refine (e.g., only fire-rated walls)
4. In the Speckle connector, select this view as the send source
5. Send — only visible elements in the view are processed

**Why this matters:**
- A full Revit model can contain 100,000+ elements
- Sending everything creates unnecessarily large Speckle versions
- View-based filtering lets you control EXACTLY what crosses the boundary
- Different views can feed different Speckle branches (structural, architectural, MEP)

---

## Example 5: Update-in-Place Workflow

Use this pattern for iterative design workflows where Revit receives updated data from the same Speckle source.

**Round 1 — Initial receive:**
1. Receive from Speckle branch "structural-model" version 1
2. Connector creates DirectShapes with `applicationId` values
3. User adds dimensions, tags, and annotations to received elements

**Round 2 — Updated receive:**
1. Structural engineer updates the model and creates version 2 on "structural-model"
2. Receive again from the same branch
3. Connector matches `applicationId` values against existing DirectShapes
4. Matched elements are updated in place — dimensions and tags are preserved
5. New elements (no matching `applicationId`) are created as new DirectShapes
6. Removed elements (present locally but not in new version) remain unchanged

**ALWAYS** use the same Speckle branch for iterative workflows.
**NEVER** delete and re-receive — this destroys all annotations and dimensions.

---

## Example 6: Reference Point Alignment for GIS Exchange

When exchanging data between Revit and GIS tools via Speckle:

```
Revit (Survey Point) → Speckle → QGIS (CRS-aware)
```

**Configuration:**
1. In Revit: Set the Survey Point to known real-world coordinates (e.g., EPSG:28992 RD New)
2. In the Speckle connector: Select "Survey Point" as the reference point
3. Send the model
4. In QGIS: Receive with the Speckle QGIS connector
5. Assign the correct CRS (EPSG:28992) to the received layer

**ALWAYS** document which reference point was used in the Speckle commit message.
**NEVER** mix reference point settings between send and receive — this causes geometry displacement equal to the distance between the reference points.
