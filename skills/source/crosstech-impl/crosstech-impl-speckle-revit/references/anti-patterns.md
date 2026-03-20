# crosstech-impl-speckle-revit — Anti-Patterns

## Anti-Pattern 1: Expecting Parametric Round-Trip Fidelity

**What people do:** Send a Revit Wall, receive it back, and expect it to be an editable Wall with join/extend behavior.

**What actually happens:** The Wall returns as a DirectShape — a frozen geometry container with no parametric intelligence. It looks like a wall but cannot join to other walls, extend to a roof, or be edited via type parameters.

**Why:** Speckle stores geometry as tessellated meshes plus metadata. The parametric definition (wall layer structure, compound structure, sweep profiles) is NOT part of the Speckle schema. Revit's parametric engine requires native family definitions that Speckle cannot serialize.

**ALWAYS** treat received geometry as reference geometry, not as editable BIM elements.
**NEVER** build a workflow that depends on round-tripped elements being parametrically editable.

---

## Anti-Pattern 2: Relying on speckle_type for BIM Classification

**What people do:** Filter Speckle objects using `speckle_type == "Objects.BuiltElements.Wall"` to find walls.

**What actually happens:** In Speckle v3, ALL BIM elements have `speckle_type = "Objects.Data.DataObject"`. The filter matches nothing or everything.

**Why:** Speckle v3 moved from typed classes to generic DataObject containers. BIM semantics are stored in the `properties` dictionary, not in the type discriminator.

**ALWAYS** check `obj.properties.get("category")` for BIM classification in Speckle v3.
**NEVER** rely on `speckle_type` to distinguish BIM element types in v3.

```python
# WRONG — v3 returns nothing
walls = [o for o in objects if o.speckle_type == "Objects.BuiltElements.Wall"]

# CORRECT — v3 pattern
walls = [o for o in objects if (getattr(o, "properties", {}) or {}).get("category") == "Walls"]
```

---

## Anti-Pattern 3: Ignoring Reference Point Configuration

**What people do:** Send from Revit using "Internal Origin" and receive in QGIS without CRS configuration.

**What actually happens:** Geometry appears at the wrong location — potentially hundreds of kilometers from the actual building site. In extreme cases, geometry appears at (0,0) in an unknown coordinate system.

**Why:** Revit's Internal Origin has no real-world meaning. The Survey Point ties the model to real-world coordinates. Without matching reference point settings, the coordinate transformation is undefined.

**ALWAYS** use "Survey Point" when exchanging with GIS tools.
**ALWAYS** document the reference point setting in the Speckle commit message.
**NEVER** assume coordinates are correct without verifying the reference point setting on both send and receive sides.

---

## Anti-Pattern 4: Sending the Entire Model Without View Filtering

**What people do:** Send all elements from a large Revit model (50,000+ elements) to Speckle.

**What actually happens:** The send operation takes 30+ minutes, creates a massive Speckle version (gigabytes), and the receiving application chokes on the data volume. Some connectors time out or crash.

**Why:** Full Revit models contain detail items, annotations, revision clouds, viewports, and other non-BIM elements that are irrelevant for cross-technology exchange. Each element is individually serialized and hashed.

**ALWAYS** create a dedicated Revit view with only the target categories visible.
**ALWAYS** apply category and view filters before sending.
**NEVER** send "Everything" from a production Revit model without filtering.

---

## Anti-Pattern 5: Deleting and Re-Receiving Instead of Updating

**What people do:** Delete all previously received DirectShapes, then receive the latest version fresh.

**What actually happens:** All dimensions, tags, annotations, and detail lines attached to the deleted elements are permanently lost. The user must redo hours of documentation work.

**Why:** Revit dimensions and tags reference specific ElementIds. Deleting an element breaks all references to it. The update-in-place mechanism (via `applicationId` matching) preserves ElementIds, which keeps dimensions and tags intact.

**ALWAYS** re-receive from the same Speckle branch — the connector updates existing elements.
**NEVER** manually delete received elements before re-receiving.

---

## Anti-Pattern 6: Mixing .NET Framework and .NET 8 Connector Versions

**What people do:** Install a Speckle connector built for .NET Framework 4.8 in Revit 2025 (which uses .NET 8), or vice versa.

**What actually happens:** The connector fails to load. Revit shows no Speckle tab, or the connector crashes immediately on startup with assembly loading errors.

**Why:** Revit 2025 switched from .NET Framework 4.8 to .NET 8.0. The two runtimes are NOT compatible. Assemblies compiled for one runtime NEVER load in the other.

**ALWAYS** download the connector version matching your Revit version:
- Revit 2022-2024 → .NET Framework 4.8 connector
- Revit 2025-2026 → .NET 8.0 connector

**NEVER** manually copy connector DLLs between Revit version folders.

---

## Anti-Pattern 7: Assuming Hosted Element Relationships Survive

**What people do:** Send a Revit model with doors hosted in walls, expecting the door-wall relationship to persist through Speckle.

**What actually happens:** On the Speckle side, doors and walls are separate Base objects with no explicit host-hosted link. On re-receive, doors are created as standalone DirectShapes positioned near (but not hosted in) wall DirectShapes.

**Why:** Speckle's object model does not have a concept of "hosted elements." The spatial relationship is preserved (the door mesh is positioned in the wall opening), but the Revit API relationship (`FamilyInstance.Host`) is lost.

**ALWAYS** treat received doors/windows as independent geometry positioned at the correct location.
**NEVER** build workflows that depend on querying `FamilyInstance.Host` on received elements.

---

## Anti-Pattern 8: Sending Without Committing (Orphaned Objects)

**What people do:** Call `Operations.Send()` but forget to call `client.CommitCreate()` (or `client.version.create()` in v3).

**What actually happens:** The objects are uploaded to the Speckle server but are not referenced by any version. They become orphaned objects — stored in the database but effectively irretrievable through the UI or standard API queries.

**Why:** Speckle separates object storage from version management. `Send` writes objects; `CommitCreate` creates a version that references those objects. Without the version, there is no entry point to find the objects.

**ALWAYS** create a version/commit immediately after a successful send.
**NEVER** treat `Operations.Send()` as a complete operation — it is only half the workflow.

```csharp
// WRONG — objects are orphaned
string objectId = await Operations.Send(root, new[] { transport });
// Missing: CommitCreate

// CORRECT — objects are findable via the version
string objectId = await Operations.Send(root, new[] { transport });
await client.CommitCreate(new CommitCreateInput
{
    streamId = streamId,
    objectId = objectId,
    branchName = "main",
    message = "Wall export"
});
```

---

## Anti-Pattern 9: Hardcoding Revit Internal Units

**What people do:** Set parameter values in millimeters or meters without converting to Revit's internal unit system.

**What actually happens:** A wall intended to be 200mm wide becomes 200 feet wide (60,960mm), or a 3-meter offset becomes 3 feet (914mm).

**Why:** Revit's internal unit for length is ALWAYS feet. ALL numeric parameter values passed through the Revit API MUST be in feet. Speckle stores values in the unit system of the source application, so conversion is required on receive.

**ALWAYS** convert units when setting Revit parameters programmatically:
- Millimeters to feet: `value_mm / 304.8`
- Meters to feet: `value_m / 0.3048`

**NEVER** pass Speckle parameter values directly to Revit API `Parameter.Set()` without unit conversion.

---

## Anti-Pattern 10: Using Automatic Type Mapping for Non-Revit Sources

**What people do:** Receive a model originating from Blender or Rhino using the "Automatic" type mapping mode.

**What actually happens:** No types match because Blender/Rhino objects do not have Revit-style Category/Family/Type naming. Everything falls back to DirectShape in the "Generic Models" category, losing any category-based organization.

**Why:** Automatic type mapping relies on matching string names (Category, Family, Type) against the Revit family library. Non-Revit sources use entirely different naming conventions.

**ALWAYS** use "Manual" type mapping when receiving from non-Revit sources.
**ALWAYS** define explicit category mappings before receiving cross-platform data.
