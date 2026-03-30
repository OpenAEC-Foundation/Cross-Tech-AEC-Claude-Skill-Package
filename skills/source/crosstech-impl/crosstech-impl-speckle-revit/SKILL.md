---
name: crosstech-impl-speckle-revit
description: >
  Use when sending or receiving BIM elements between Speckle and Revit.
  Prevents the common mistake of expecting perfect round-trip fidelity or
  losing Revit family type information during Speckle conversion.
  Covers Speckle Revit connector, family type mapping, parameter synchronization,
  view-based filtering, DirectShape receive, and round-trip data integrity.
  Keywords: Speckle, Revit, connector, BIM elements, family types, parameters,
  DirectShape, round-trip, send receive, Revit API, share Revit model,
  Revit to web, sync Revit data, collaborate on BIM.
license: MIT
compatibility: "Designed for Claude Code. Covers Speckle 2.x/3.x, Revit 2022-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-impl-speckle-revit

## Quick Reference

### Side A: Speckle (Base Objects)

| Concept | Description |
|---------|-------------|
| `Base` | Root class for all Speckle objects; hash-based immutable identity |
| `speckle_type` | Type discriminator — in v3 ALWAYS `Objects.Data.DataObject` for BIM elements |
| `displayValue` | Mesh geometry for rendering; the atomic visual unit |
| `parameters` | Nested dictionary holding Type Parameters and Instance Parameters |
| `applicationId` | Secondary ID from the host application; enables update-in-place |
| `Collection` | Hierarchical grouping with `name`, `collectionType`, `elements` |
| Proxy Collections | `levelProxies`, `colorProxies`, `renderMaterialProxies` at root level |

### Side B: Revit (Families, Parameters, Elements)

| Concept | Description |
|---------|-------------|
| Family | Template defining geometry and behavior (e.g., `M_Single-Flush`) |
| Family Type | Specific parametric variant of a Family (e.g., `0915 x 2134mm`) |
| System Family | Built-in families: Walls, Floors, Roofs, Ceilings, Stairs |
| Instance Parameter | Per-element values (Base Offset, Mark, Comments) |
| Type Parameter | Shared across all instances of a type (Width, Fire Rating) |
| DirectShape | Geometry-only container; no parametric intelligence |
| Workset | Collaboration partition; NOT preserved through Speckle |
| ElementId | Revit's internal integer identifier; unstable across sessions |

### The Bridge: Speckle Revit Connector

| Aspect | Detail |
|--------|--------|
| Directionality | Bidirectional with **asymmetric fidelity** |
| Send granularity | View-based, selection-based, category-based, or workset-based |
| Receive output | DirectShapes (default) or mapped native families |
| Parameter handling | ALL parameters extracted; nested Type/Instance structure |
| Update mechanism | `applicationId` matching enables update-in-place |
| Reference points | Internal Origin, Project Base Point, or Survey Point |

### Critical Warnings

**NEVER** expect round-trip parametric fidelity — elements sent from Revit return as DirectShapes, NOT as editable native families.

**NEVER** rely on `speckle_type` for BIM classification in Speckle v3 — it is ALWAYS `Objects.Data.DataObject`. Inspect `properties.category` instead.

**NEVER** assume hosted element relationships survive — doors/windows lose their wall-host association after Speckle transport.

**NEVER** omit reference point configuration — mismatched reference points between send and receive cause geometry displacement.

**ALWAYS** verify `applicationId` values before re-receiving — they control whether elements are updated or duplicated.

**ALWAYS** set the Detail Level (Low/Medium/High) before receiving — it controls tessellation quality of DirectShape geometry.

---

## Send Pipeline: Revit to Speckle

### Stage 1: Element Selection

Four selection strategies exist. ALWAYS choose based on project needs:

| Strategy | Use When | API Entry Point |
|----------|----------|-----------------|
| View-based | Sending visible elements from a specific view | Active view filter |
| Selection set | Sending hand-picked elements | Current selection |
| Category filter | Sending all elements of specific categories | Category list |
| Workset filter | Sending by collaboration partition | Workset list |

### Stage 2: Element Unpacking

Complex elements are expanded into atomic objects:
- Curtain walls decompose into panels + mullions
- Stairs decompose into runs + landings + railings
- Groups are expanded into individual members

### Stage 3: Conversion

The `RevitRootObjectBuilder` converts each Revit element:

1. Reads element geometry via Revit API `GeometryElement`
2. Extracts ALL parameters (Type + Instance) into nested dictionary
3. Resolves Family, FamilyType, and Category names
4. Generates `displayValue` meshes at the configured Detail Level
5. Assigns `applicationId` from Revit's `UniqueId` property

### Stage 4: Caching

The `RevitToSpeckleCacheSingleton` prevents redundant conversion. Elements with unchanged geometry and parameters reuse cached Speckle objects. This is critical for large models (10,000+ elements).

### Stage 5: Transport

Serialized objects are sent to the selected transport (ServerTransport for cloud, SQLiteTransport for local). The root object hash becomes the version reference.

---

## Receive Pipeline: Speckle to Revit

### Stage 1: Object Reception

Download and deserialize the root object and all children from the transport.

### Stage 2: Type Mapping

The connector matches Speckle objects to Revit families/types:

| Mapping Mode | Behavior |
|-------------|----------|
| **Automatic** | Matches by Category + Family + Type name strings |
| **Manual** | Presents a mapping table for user assignment |

**ALWAYS** use manual mapping when receiving from non-Revit sources (Blender, Rhino, ArchiCAD) — automatic mapping relies on Revit-specific naming conventions that other tools do not produce.

### Stage 3: DirectShape Creation

When no matching native family exists, elements are created as DirectShapes:
- DirectShapes accept arbitrary solid/mesh geometry
- They belong to a Revit category (Walls, Generic Models, etc.)
- They have NO parametric behavior — geometry is frozen
- They CAN receive parameter values as custom shared parameters

### Stage 4: Material Application

The `RevitMaterialBaker` creates or matches Revit materials:
- Maps Speckle `RenderMaterial` to Revit `Material` class
- Basic properties transfer: diffuse color, transparency, metalness
- Complex shader graphs do NOT transfer — only base color values

### Stage 5: Group and Hierarchy

The `RevitGroupBaker` recreates organizational structure:
- Speckle Collections become Revit Groups
- Level assignments from `levelProxies` are restored
- Spatial hierarchy is approximated but NOT identical to original

### Stage 6: Transaction Commit

ALL Revit modifications are wrapped in a single `Transaction`:

```csharp
// Speckle connector internal pattern
using (Transaction t = new Transaction(doc, "Speckle Receive"))
{
    t.Start();
    // ... create DirectShapes, apply materials, set parameters ...
    t.Commit();
}
```

If any element fails, the entire transaction rolls back. Check the Speckle log for conversion warnings.

---

## Round-Trip Data Integrity

### What SURVIVES Revit to Speckle to Revit

| Data | Fidelity | Notes |
|------|----------|-------|
| Geometry shape | High | Tessellated mesh; Detail Level affects quality |
| Parameter values | High | Both Type and Instance; stored in nested dict |
| Material assignments | Medium | Basic color/opacity; no PBR shader graphs |
| Object grouping | Medium | Collections approximate Revit groups |
| applicationId | Exact | Enables update-in-place on re-receive |
| Category assignment | High | Revit category preserved as string |

### What is LOST or DEGRADED

| Data | Impact | Why |
|------|--------|-----|
| Parametric intelligence | **Critical** | Elements return as DirectShapes, not editable families |
| System family behavior | **Critical** | Walls, floors, roofs lose join/extend/attach logic |
| Hosted element relationships | **High** | Door-in-wall, window-in-wall links are flattened |
| Workset assignments | **High** | Not tracked through Speckle transport |
| Design options | **High** | Not represented in Speckle schema |
| Schedule grouping | **Medium** | Parameter grouping for schedules is lost |
| Phase information | **Medium** | Created/demolished phase data not preserved |
| Detailed edge profiles | **Low** | Swept profiles simplified to mesh |

### Update-in-Place Behavior

When receiving from the same source a second time:

1. Connector reads `applicationId` of each incoming object
2. Matches against `applicationId` values of existing DirectShapes in the project
3. **Match found**: Updates geometry and parameters of existing element
4. **No match**: Creates a new DirectShape

This preserves dimensions, tags, and annotations attached to previously received elements. ALWAYS use the same Speckle model/branch for iterative workflows.

---

## Reference Point Configuration

| Setting | Coordinate Origin | Use When |
|---------|-------------------|----------|
| Internal Origin | Revit's absolute (0,0,0) | Default; intra-Revit exchange |
| Project Base Point | User-defined project origin | Aligning with site coordinates |
| Survey Point | Real-world survey coordinates | Exchanging with GIS/geospatial tools |

**ALWAYS** use Survey Point when exchanging data between Revit and QGIS/GIS tools via Speckle.

**ALWAYS** use the same reference point setting for both send and receive operations in a round-trip workflow.

---

## Speckle .NET SDK: Programmatic Access

For automated pipelines bypassing the UI connector:

```csharp
using Speckle.Core.Api;
using Speckle.Core.Credentials;
using Speckle.Core.Transports;
using Speckle.Core.Models;

// Authenticate
var account = AccountManager.GetDefaultAccount();
var client = new Client(account);

// Create transport
var transport = new ServerTransport(account, "stream-id-here");

// Send a Base object
var myObject = new Base();
myObject["category"] = "Walls";
myObject["parameters"] = parametersDictionary;

string objectId = await Operations.Send(myObject, new[] { transport });

// Create a version (commit)
var commitId = await client.CommitCreate(new CommitCreateInput
{
    streamId = "stream-id-here",
    objectId = objectId,
    branchName = "main",
    message = "Automated wall export"
});
```

---

## Linked Model Support

The connector supports sending linked Revit models:
- Each linked model creates a separate `DocumentToConvert` context
- A transformation matrix positions the linked model relative to the host
- `applicationId` values are modified with a transform hash to distinguish instances of the same linked model at different positions
- **ALWAYS** verify that linked model positions are correct after receiving — transformation stacking can introduce drift

---

## Version Compatibility Matrix

| Revit Version | .NET Runtime | WebView Technology | Speckle Connector |
|--------------|-------------|-------------------|-------------------|
| 2022-2024 | .NET Framework 4.8 | CefSharp | Speckle 2.x / 3.x |
| 2025-2026 | .NET 8.0 | WebView2 | Speckle 3.x |

**ALWAYS** match the connector version to the Revit version — a .NET 4.8 connector NEVER works in Revit 2025+.

---

## Reference Links

- [references/methods.md](references/methods.md) — API methods for send/receive, type mapping, parameter extraction
- [references/examples.md](references/examples.md) — Working code examples for common Speckle-Revit workflows
- [references/anti-patterns.md](references/anti-patterns.md) — What NOT to do at the Speckle-Revit boundary

### Official Sources

- https://speckle.guide/user/revit.html
- https://speckle.systems/tutorials/
- https://github.com/specklesystems/speckle-sharp
- https://github.com/specklesystems/speckle-sharp-connectors
- https://speckle.guide/dev/dotnet.html
