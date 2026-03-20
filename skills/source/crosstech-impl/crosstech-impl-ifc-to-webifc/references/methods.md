# Methods Reference: IfcOpenShell API vs web-ifc IfcAPI

## File Lifecycle Methods

### IfcOpenShell (Python)

| Method | Signature | Purpose |
|--------|-----------|---------|
| `ifcopenshell.open()` | `open(path: str) -> file` | Open IFC from file path |
| `ifcopenshell.file()` | `file(schema: str = "IFC4") -> file` | Create blank IFC model |
| `model.write()` | `write(path: str) -> None` | Save to disk |
| `model.schema` | Property: `str` | Returns "IFC2X3", "IFC4", or "IFC4X3" |

### web-ifc (JavaScript/TypeScript)

| Method | Signature | Purpose |
|--------|-----------|---------|
| `new IfcAPI()` | Constructor | Create API instance |
| `api.Init()` | `() -> Promise<void>` | Initialize WASM -- **MUST call first** |
| `api.OpenModel()` | `(data: Uint8Array, settings?: LoaderSettings) -> number` | Load IFC, returns modelID |
| `api.OpenModelFromCallback()` | `(callback: Function) -> number` | Stream-load for large files |
| `api.CreateModel()` | `(model: NewIfcModel, settings?: LoaderSettings) -> number` | Create empty model |
| `api.CloseModel()` | `(modelID: number) -> void` | Free ALL WASM memory for model |
| `api.IsModelOpen()` | `(modelID: number) -> boolean` | Check model status |
| `api.GetModelSchema()` | `(modelID: number) -> string` | Get IFC schema version |
| `api.SaveModel()` | `(modelID: number) -> Uint8Array` | Export to IFC buffer |

---

## Entity Query Methods

### IfcOpenShell

| Method | Signature | Returns |
|--------|-----------|---------|
| `model.by_type()` | `(type: str, include_subtypes=True) -> list[entity_instance]` | All entities of given IFC class |
| `model.by_id()` | `(id: int) -> entity_instance` | Entity by STEP numerical ID |
| `model.by_guid()` | `(guid: str) -> entity_instance` | Entity by 22-char GlobalId |
| `model.traverse()` | `(inst, max_levels=None) -> list` | All referenced instances recursively |
| `model.get_inverse()` | `(inst) -> set` | All entities referencing this entity |

### web-ifc

| Method | Signature | Returns |
|--------|-----------|---------|
| `api.GetLine()` | `(modelID, expressID, flatten?, inverse?, inversePropKey?) -> object` | Entity by EXPRESS ID |
| `api.GetAllLines()` | `(modelID) -> Vector<number>` | All entity IDs |
| `api.GetLineIDsWithType()` | `(modelID, typeID) -> Vector<number>` | Entity IDs by IFC type constant |

**`GetLine` parameters explained**:
- `flatten` (default: false): When true, recursively resolves all `Handle` references to full objects. When false, nested entities appear as `{expressID: N, type: T}`.
- `inverse` (default: false): When true, includes inverse attributes (entities that reference this entity).
- `inversePropKey` (optional): Filter inverse results to a specific property name.

---

## Property Access Methods

### IfcOpenShell -- High-Level Utilities

```python
import ifcopenshell.util.element

# RECOMMENDED: get all properties merged from occurrence + type
psets = ifcopenshell.util.element.get_psets(element)
# Returns dict: {"Pset_WallCommon": {"IsExternal": True, ...}, "Qto_...": {...}}

# Properties only (exclude quantities)
psets = ifcopenshell.util.element.get_psets(element, psets_only=True)

# Quantities only
qtos = ifcopenshell.util.element.get_psets(element, qtos_only=True)

# Get type object
element_type = ifcopenshell.util.element.get_type(element)

# Get spatial container
container = ifcopenshell.util.element.get_container(element)

# Get material
material = ifcopenshell.util.element.get_material(element)
```

### IfcOpenShell -- Low-Level Access

```python
# Direct attribute access
wall.Name           # "Basic Wall:Generic - 200mm"
wall.GlobalId       # "2XQ$n5SLP5MBLyL442paFx"
wall.is_a()         # "IfcWall"
wall.is_a("IfcProduct")  # True (checks inheritance)
wall.id()           # 122 (STEP numerical ID)
wall.get_info()     # {"id": 122, "type": "IfcWall", "GlobalId": "...", ...}

# Manual relationship traversal
for rel in wall.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        pset = rel.RelatingPropertyDefinition
        if pset.is_a("IfcPropertySet"):
            for prop in pset.HasProperties:
                print(f"{prop.Name}: {prop.NominalValue.wrappedValue}")
```

### web-ifc -- Properties Helper

```javascript
const props = api.properties;

// Element properties
const item = await props.getItemProperties(modelID, expressID);

// Property sets (resolved references when flatten=true)
const psets = await props.getPropertySets(modelID, expressID, true);

// Materials
const materials = await props.getMaterialsProperties(modelID, expressID);

// Spatial structure tree
const structure = await props.getSpatialStructure(modelID);
```

### web-ifc -- Manual Access

```javascript
// Default: nested references are Handle objects
const wall = api.GetLine(modelID, expressID);
wall.Name.value;           // "Basic Wall:Generic - 200mm"
wall.GlobalId.value;       // "2XQ$n5SLP5MBLyL442paFx"

// Flattened: all references resolved recursively
const wallFlat = api.GetLine(modelID, expressID, true);

// With inverse attributes
const wallInv = api.GetLine(modelID, expressID, false, true);
// Includes entities that reference this wall
```

---

## Geometry Methods

### IfcOpenShell

```python
import ifcopenshell.geom

# Settings
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)   # World coordinates
settings.set(settings.APPLY_DEFAULT_MATERIALS, True)

# Single element
shape = ifcopenshell.geom.create_shape(settings, element)
verts = shape.geometry.verts   # Flat: [x1,y1,z1, x2,y2,z2, ...] Z-UP
faces = shape.geometry.faces   # Flat: [i1,i2,i3, ...]
normals = shape.geometry.normals  # Flat: [nx1,ny1,nz1, ...]

# Batch iterator (multi-process)
import multiprocessing
iterator = ifcopenshell.geom.iterator(
    settings, model, multiprocessing.cpu_count()
)
if iterator.initialize():
    while True:
        shape = iterator.get()
        # shape.id, shape.geometry.verts, shape.geometry.faces
        if not iterator.next():
            break
```

### web-ifc

```javascript
// Single element mesh
const flatMesh = api.GetFlatMesh(modelID, expressID);
const placedGeometries = flatMesh.geometries;

for (let i = 0; i < placedGeometries.size(); i++) {
  const pg = placedGeometries.get(i);
  const geom = api.GetGeometry(modelID, pg.geometryExpressID);

  // 6 floats per vertex: x, y, z, nx, ny, nz (Y-UP)
  const verts = api.GetVertexArray(geom.GetVertexData(), geom.GetVertexDataSize());
  const indices = api.GetIndexArray(geom.GetIndexData(), geom.GetIndexDataSize());

  // Placement transform (4x4 flat array)
  const transform = pg.flatTransformation;

  // Color (RGBA)
  const color = pg.color; // {x: r, y: g, z: b, w: a}
}

// Streaming (memory-efficient for large models)
api.StreamAllMeshes(modelID, (flatMesh) => {
  // Process one element at a time
});

// Stream specific elements only
api.StreamMeshes(modelID, [id1, id2, id3], (flatMesh) => {
  // Process selected elements
});

// Load all at once (ONLY for small models)
const allMeshes = api.LoadAllGeometry(modelID);
```

---

## Unit and Placement Methods

### IfcOpenShell

```python
import ifcopenshell.util.unit
import ifcopenshell.util.placement

# Unit scale: conversion factor to SI meters
scale = ifcopenshell.util.unit.calculate_unit_scale(model)
meters = ifc_value * scale

# Get 4x4 transformation matrix for element placement
matrix = ifcopenshell.util.placement.get_local_placement(element.ObjectPlacement)
# matrix[:,3][:3] gives XYZ position in model coordinates
```

### web-ifc

```javascript
// Coordination matrix (transforms from local to global)
const coordMatrix = api.GetCoordinationMatrix(modelID);
// Returns Float64Array (4x4 matrix, column-major)

// Per-element placement is embedded in FlatMesh.geometries[i].flatTransformation
```

---

## LoaderSettings (web-ifc)

| Setting | Type | Default | Purpose |
|---------|------|---------|---------|
| `COORDINATE_TO_ORIGIN` | boolean | true | Center geometry at origin, convert to Y-up |
| `CIRCLE_SEGMENTS` | number | -- | Tessellation detail for circular geometry |
| `MEMORY_LIMIT` | number | ~2GB | Maximum WASM heap allocation |
| `TAPE_SIZE` | number | 67108864 (64MB) | Internal read buffer size |
| `LINEWRITER_BUFFER` | number | -- | Write buffer size |

**ALWAYS** increase `TAPE_SIZE` for files larger than 50MB. The default 64MB buffer causes parsing failures on large models.

---

## Type Constant Mapping (web-ifc)

web-ifc uses numeric constants for IFC types. Import from `web-ifc`:

```javascript
import { IFCWALL, IFCSLAB, IFCBEAM, IFCCOLUMN, IFCDOOR, IFCWINDOW,
         IFCBUILDINGSTOREY, IFCBUILDING, IFCSITE, IFCPROJECT,
         IFCPROPERTYSET, IFCRELDEFINESBYPROPERTIES } from 'web-ifc';

// Usage
const wallIDs = api.GetLineIDsWithType(modelID, IFCWALL);
```

IfcOpenShell uses string names directly:
```python
walls = model.by_type("IfcWall")
```
