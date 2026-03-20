# crosstech-impl-freecad-ifc-bridge — Methods Reference

## FreeCAD BIM/Arch API

### Arch Module — BIM Object Creation

| Method | Signature | Returns | Notes |
|--------|-----------|---------|-------|
| `Arch.makeWall` | `makeWall(baseobj=None, length=None, width=None, height=None, align="Center", face=None, name="Wall")` | `Arch::Wall` | Creates parametric wall; maps to `IfcWall` |
| `Arch.makeStructure` | `makeStructure(baseobj=None, length=None, width=None, height=None, name="Structure")` | `Arch::Structure` | Generic structural element; set `IfcType` to `IfcColumn`, `IfcBeam`, or `IfcSlab` |
| `Arch.makeWindow` | `makeWindow(baseobj=None, width=None, height=None, parts=None, name="Window")` | `Arch::Window` | Creates window; set `IfcType` to `IfcWindow` or `IfcDoor` |
| `Arch.makeStairs` | `makeStairs(baseobj=None, length=None, width=None, height=None, steps=None, name="Stairs")` | `Arch::Stairs` | Maps to `IfcStair` |
| `Arch.makeRoof` | `makeRoof(baseobj=None, facenr=0, angles=None, run=None, idrel=None, thickness=None, overhang=None, name="Roof")` | `Arch::Roof` | Maps to `IfcRoof` |
| `Arch.makePipe` | `makePipe(baseobj=None, diameter=0, length=0, placement=None, name="Pipe")` | `Arch::Pipe` | Maps to `IfcPipeSegment` |
| `Arch.makeSite` | `makeSite(objectslist=None, baseobj=None, name="Site")` | `Arch::Site` | Spatial element; maps to `IfcSite` |
| `Arch.makeBuilding` | `makeBuilding(objectslist=None, baseobj=None, name="Building")` | `Arch::Building` | Spatial element; maps to `IfcBuilding` |
| `Arch.makeFloor` | `makeFloor(objectslist=None, baseobj=None, name="Floor")` | `Arch::Floor` | Spatial element; maps to `IfcBuildingStorey` |
| `Arch.makeSpace` | `makeSpace(objects=None, baseobj=None, name="Space")` | `Arch::Space` | Maps to `IfcSpace` |
| `Arch.makeEquipment` | `makeEquipment(baseobj=None, name="Equipment")` | `Arch::Equipment` | Maps to `IfcFurnishingElement` |

### Arch Module — IFC Properties

| Method | Purpose | Usage |
|--------|---------|-------|
| `obj.IfcType` | Get/set IFC entity type | `wall.IfcType = "IfcWall"` |
| `obj.IfcProperties` | Get/set custom IFC properties | Dictionary of property sets |
| `obj.IfcData` | Raw IFC data dictionary | Low-level IFC attributes |
| `obj.Label` | Human-readable name | Maps to `IfcProduct.Name` |
| `obj.Description` | Element description | Maps to `IfcProduct.Description` |

### FreeCAD Document API

| Method | Signature | Purpose |
|--------|-----------|---------|
| `FreeCAD.openDocument` | `openDocument(filepath)` | Open document (IFC or FCStd) |
| `FreeCAD.newDocument` | `newDocument(name="Unnamed")` | Create new document |
| `FreeCAD.ActiveDocument` | Property | Currently active document |
| `doc.save` | `doc.save()` | Save document (writes to IFC in NativeIFC) |
| `doc.saveAs` | `doc.saveAs(filepath)` | Save to new path |
| `doc.recompute` | `doc.recompute()` | Recompute all objects |
| `doc.Objects` | Property | List of all document objects |
| `doc.getObject` | `doc.getObject(name)` | Get object by internal name |
| `doc.getObjectsByLabel` | `doc.getObjectsByLabel(label)` | Get objects by label (list) |
| `doc.addObject` | `doc.addObject(type, name)` | Add new object to document |
| `doc.removeObject` | `doc.removeObject(name)` | Remove object from document |
| `doc.FileName` | Property | Full path of the document file |

### Part Module — Geometry Operations

| Method | Purpose | Notes |
|--------|---------|-------|
| `Part.makeBox` | Create box solid | `Part.makeBox(length, width, height)` |
| `Part.makeCompound` | Combine shapes | `Part.makeCompound([shape1, shape2])` |
| `Part.show` | Display shape | `Part.show(shape, name)` |
| `obj.Shape` | Access TopoShape | Triggers geometry computation in NativeIFC |
| `shape.exportStep` | Export to STEP | `shape.exportStep("/path/to/file.step")` |
| `shape.exportBrep` | Export to BREP | `shape.exportBrep("/path/to/file.brep")` |

## IfcOpenShell API Within FreeCAD

### Core Operations

| Operation | Code | Notes |
|-----------|------|-------|
| Open IFC file | `ifc = ifcopenshell.open(path)` | Returns `ifcopenshell.file` |
| Create new IFC | `ifc = ifcopenshell.file(schema="IFC4")` | Specify schema version |
| Get schema | `ifc.schema` | Returns `"IFC2X3"`, `"IFC4"`, or `"IFC4X3"` |
| Get by type | `ifc.by_type("IfcWall")` | Returns list of entities |
| Get by id | `ifc.by_id(42)` | Returns single entity by expressID |
| Get by guid | `ifc.by_guid("2O2Fr...")` | Returns single entity by GlobalId |
| Create entity | `ifc.createIfcWall(guid, owner, name, ...)` | Positional args match schema |
| Write file | `ifc.write(path)` | Save to disk |

### Utility Functions

| Function | Module | Purpose |
|----------|--------|---------|
| `get_psets(element)` | `ifcopenshell.util.element` | Get ALL property sets (occurrence + type) |
| `get_type(element)` | `ifcopenshell.util.element` | Get the type object for an occurrence |
| `get_container(element)` | `ifcopenshell.util.element` | Get spatial container (storey, space) |
| `get_aggregate(element)` | `ifcopenshell.util.element` | Get parent aggregate |
| `get_decomposition(element)` | `ifcopenshell.util.element` | Get child elements |
| `calculate_unit_scale(model)` | `ifcopenshell.util.unit` | Get SI unit scale factor |
| `get_local_placement(product)` | `ifcopenshell.util.placement` | Get 4x4 placement matrix |
| `create_shape(settings, product)` | `ifcopenshell.geom` | Generate geometry (BREP/mesh) |

### Geometry Settings for FreeCAD

```python
import ifcopenshell.geom

# Settings for FreeCAD (OpenCASCADE BREP output)
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_BREP_DATA, True)       # BREP for FreeCAD Part
settings.set(settings.USE_WORLD_COORDS, False)    # Apply placement separately
settings.set(settings.DISABLE_TRIANGULATION, True) # BREP, not mesh

# Settings for tessellated mesh output
mesh_settings = ifcopenshell.geom.settings()
mesh_settings.set(mesh_settings.USE_BREP_DATA, False)
mesh_settings.set(mesh_settings.USE_WORLD_COORDS, True)
```

## Legacy Import/Export Module

### importIFC Module (Legacy)

| Function | Purpose | Notes |
|----------|---------|-------|
| `importIFC.open(filepath)` | Open IFC as FreeCAD doc | Legacy method; prefer `FreeCAD.openDocument` |
| `importIFC.insert(filepath, docname)` | Insert IFC into existing doc | Merges IFC objects into document |
| `importIFC.export(objectslist, filepath)` | Export objects to IFC | Creates new IFC file from FreeCAD objects |

### Import Preferences (Edit > Preferences > Import-Export > IFC)

| Setting | Default | Recommended | Effect |
|---------|---------|-------------|--------|
| Import full model | True | True | Load all elements vs. selected types only |
| Create clones for shared types | True | True | Memory optimization for repeated elements |
| Separate openings | True | True | Create separate objects for voids |
| Prefix with type number | False | False | Add expressID prefix to object names |
| Merge materials | False | True | Combine identical materials |
| Import FreeCAD properties | True | True | Preserve round-trip custom properties |
| Fit view on import | True | True | Zoom to fit imported model |
