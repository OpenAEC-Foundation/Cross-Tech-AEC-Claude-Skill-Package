# crosstech-core-ifc-schema-bridge: Working Code Examples

All examples verified against research document vooronderzoek-ifc-ecosystem.md
and official documentation:
- https://docs.ifcopenshell.org/
- https://github.com/ThatOpen/engine_web-ifc
- https://docs.speckle.systems/developers/data-schema/connectors/ifc-schema

---

## Example 1: Read Properties — IfcOpenShell (Python)

```python
# IfcOpenShell 0.8.x — read all properties from a wall (occurrence + type combined)
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")

# ALWAYS check schema first
print(f"Schema: {model.schema}")  # "IFC2X3", "IFC4", "IFC4X3"

wall = model.by_type("IfcWall")[0]

# get_psets() returns properties from BOTH the occurrence AND its type
psets = ifcopenshell.util.element.get_psets(wall)
# Returns: {
#   "Pset_WallCommon": {"IsExternal": True, "LoadBearing": False, ...},
#   "Qto_WallBaseQuantities": {"Length": 5.0, "Height": 3.0, ...},
#   "MyCustomPset": {"CustomProp": "value"}
# }

# Access a specific property
is_external = psets.get("Pset_WallCommon", {}).get("IsExternal")

# Get properties only (exclude quantity sets)
props_only = ifcopenshell.util.element.get_psets(wall, psets_only=True)

# Get quantities only (exclude property sets)
qtos_only = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
```

---

## Example 2: Read Properties — web-ifc (TypeScript)

```typescript
// web-ifc 0.0.66+ — read properties from a wall (manual traversal required)
import * as WebIFC from "web-ifc";

const ifcApi = new WebIFC.IfcAPI();
await ifcApi.Init();

const data = new Uint8Array(arrayBuffer);
const modelID = ifcApi.OpenModel(data);

// ALWAYS check schema first
const schema = ifcApi.GetModelSchema(modelID);
console.log(`Schema: ${schema}`);

// Get all IfcRelDefinesByProperties to find property sets for a given element
function getPropertySets(
  ifcApi: WebIFC.IfcAPI,
  modelID: number,
  elementExpressID: number
): Record<string, Record<string, any>> {
  const result: Record<string, Record<string, any>> = {};

  // Get all IfcRelDefinesByProperties entities
  const allLines = ifcApi.GetAllLines(modelID);
  for (const lineID of allLines) {
    const typeCode = ifcApi.GetLineType(modelID, lineID);
    if (typeCode !== WebIFC.IFCRELDEFINESBYPROPERTIES) continue;

    const rel = ifcApi.GetLine(modelID, lineID, true);
    // Check if this relationship targets our element
    const relatedObjects = rel.RelatedObjects;
    const isRelated = relatedObjects.some(
      (obj: any) => obj.expressID === elementExpressID
    );
    if (!isRelated) continue;

    // Get the property set
    const psetRef = rel.RelatingPropertyDefinition;
    const pset = ifcApi.GetLine(modelID, psetRef.expressID, true);

    if (pset.type === WebIFC.IFCPROPERTYSET) {
      const psetName = pset.Name.value;
      result[psetName] = {};
      for (const propRef of pset.HasProperties) {
        const prop = ifcApi.GetLine(modelID, propRef.expressID, true);
        if (prop.NominalValue) {
          result[psetName][prop.Name.value] = prop.NominalValue.value;
        }
      }
    }
  }
  return result;
}

// Usage
const wallExpressID = 122; // known expressID
const psets = getPropertySets(ifcApi, modelID, wallExpressID);
console.log(psets);

ifcApi.CloseModel(modelID);
```

---

## Example 3: Spatial Structure Traversal — IfcOpenShell

```python
# IfcOpenShell 0.8.x — traverse the full spatial hierarchy
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")

# Get the project (ALWAYS exactly one per file)
project = model.by_type("IfcProject")[0]

# Traverse spatial hierarchy: Project → Site → Building → Storey
def print_spatial_tree(element, indent=0):
    print(f"{'  ' * indent}{element.is_a()} — {element.Name}")

    # Get children via IfcRelAggregates
    children = ifcopenshell.util.element.get_decomposition(element)
    for child in children:
        print_spatial_tree(child, indent + 1)

    # Get contained elements (only for spatial elements)
    if element.is_a("IfcSpatialElement") or element.is_a("IfcSpatialStructureElement"):
        for rel in getattr(element, "ContainsElements", []):
            for elem in rel.RelatedElements:
                print(f"{'  ' * (indent + 1)}[contained] {elem.is_a()} — {elem.Name}")

print_spatial_tree(project)
# Output:
# IfcProject — My Project
#   IfcSite — Default Site
#     IfcBuilding — Main Building
#       IfcBuildingStorey — Ground Floor
#         [contained] IfcWall — Wall 001
#         [contained] IfcSlab — Floor Slab
#       IfcBuildingStorey — First Floor
#         [contained] IfcWall — Wall 101
```

---

## Example 4: Geometry Extraction — IfcOpenShell

```python
# IfcOpenShell 0.8.x — extract tessellated geometry for cross-tool use
import ifcopenshell
import ifcopenshell.geom
import ifcopenshell.util.unit
import multiprocessing

model = ifcopenshell.open("model.ifc")

# ALWAYS get unit scale before processing geometry
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
print(f"Unit scale to meters: {unit_scale}")

# Geometry settings
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)  # Apply placement transforms

# Process single element
wall = model.by_type("IfcWall")[0]
shape = ifcopenshell.geom.create_shape(settings, wall)

# Extract vertex/face data (flat arrays)
verts = shape.geometry.verts      # [x1,y1,z1, x2,y2,z2, ...]
faces = shape.geometry.faces      # [i1,i2,i3, ...]  (triangle indices)

# Convert to list of tuples
vertices = [(verts[i], verts[i+1], verts[i+2]) for i in range(0, len(verts), 3)]
triangles = [(faces[i], faces[i+1], faces[i+2]) for i in range(0, len(faces), 3)]

# Apply unit scale if needed (when USE_WORLD_COORDS is True, coordinates are in file units)
vertices_meters = [(v[0] * unit_scale, v[1] * unit_scale, v[2] * unit_scale) for v in vertices]

# Batch process all products (for large models)
iterator = ifcopenshell.geom.iterator(settings, model, multiprocessing.cpu_count())
if iterator.initialize():
    while True:
        shape = iterator.get()
        element = model.by_id(shape.id)
        print(f"{element.is_a()} — {element.Name}: {len(shape.geometry.verts) // 3} vertices")
        if not iterator.next():
            break
```

---

## Example 5: Geometry Extraction — web-ifc

```typescript
// web-ifc 0.0.66+ — extract geometry for Three.js
import * as WebIFC from "web-ifc";

const ifcApi = new WebIFC.IfcAPI();
await ifcApi.Init();
const modelID = ifcApi.OpenModel(data);

// Get coordination matrix (global transform)
const coordMatrix = ifcApi.GetCoordinationMatrix(modelID);

// Stream all meshes (memory-efficient for large models)
ifcApi.StreamAllMeshes(modelID, (flatMesh: WebIFC.FlatMesh) => {
  const expressID = flatMesh.expressID;
  const element = ifcApi.GetLine(modelID, expressID);

  for (const placedGeom of flatMesh.geometries) {
    const geometry = ifcApi.GetGeometry(modelID, placedGeom.geometryExpressID);

    // Vertex data (Float32Array)
    const vertexDataPtr = geometry.GetVertexData();
    const vertexDataSize = geometry.GetVertexDataSize();
    const vertexArray = new Float32Array(
      ifcApi.wasmModule.HEAPF32.buffer,
      vertexDataPtr,
      vertexDataSize / 4
    );

    // Index data (Uint32Array)
    const indexDataPtr = geometry.GetIndexData();
    const indexDataSize = geometry.GetIndexDataSize();
    const indexArray = new Uint32Array(
      ifcApi.wasmModule.HEAPU32.buffer,
      indexDataPtr,
      indexDataSize / 4
    );

    // The 4x4 transform matrix for this placement
    const transform = placedGeom.flatTransformation;

    // Clean up WASM memory
    geometry.delete();
  }
});

ifcApi.CloseModel(modelID);
```

---

## Example 6: Type Object Access — IfcOpenShell

```python
# IfcOpenShell 0.8.x — access type objects and their properties
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# Get the type object (IfcWallType)
wall_type = ifcopenshell.util.element.get_type(wall)
if wall_type:
    print(f"Type: {wall_type.Name}")  # e.g., "Basic Wall:Generic - 200mm"

    # Type-level properties (shared by all instances of this type)
    type_psets = ifcopenshell.util.element.get_psets(wall_type)

    # Occurrence-level properties (specific to this wall)
    occurrence_psets = ifcopenshell.util.element.get_psets(wall)

    # IMPORTANT: get_psets(wall) already merges type + occurrence properties.
    # To get ONLY occurrence-level properties:
    occurrence_only = {}
    for rel in wall.IsDefinedBy:
        if rel.is_a("IfcRelDefinesByProperties"):
            pset = rel.RelatingPropertyDefinition
            if pset.is_a("IfcPropertySet"):
                props = {}
                for prop in pset.HasProperties:
                    if prop.is_a("IfcPropertySingleValue"):
                        props[prop.Name] = prop.NominalValue.wrappedValue if prop.NominalValue else None
                occurrence_only[pset.Name] = props
```

---

## Example 7: Quantity Set Access — IfcOpenShell

```python
# IfcOpenShell 0.8.x — access quantity sets
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("model.ifc")
wall = model.by_type("IfcWall")[0]

# get_psets() returns both property sets AND quantity sets
all_data = ifcopenshell.util.element.get_psets(wall)

# Filter to quantity sets only
qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
# Returns: {"Qto_WallBaseQuantities": {"Length": 5.0, "Height": 3.0, "GrossVolume": 3.0, ...}}

# Direct access to quantity entities
for rel in wall.IsDefinedBy:
    if rel.is_a("IfcRelDefinesByProperties"):
        qto = rel.RelatingPropertyDefinition
        if qto.is_a("IfcElementQuantity"):
            print(f"Quantity set: {qto.Name}")
            for quantity in qto.Quantities:
                qtype = quantity.is_a()
                if qtype == "IfcQuantityLength":
                    print(f"  {quantity.Name}: {quantity.LengthValue} (length)")
                elif qtype == "IfcQuantityArea":
                    print(f"  {quantity.Name}: {quantity.AreaValue} (area)")
                elif qtype == "IfcQuantityVolume":
                    print(f"  {quantity.Name}: {quantity.VolumeValue} (volume)")
                elif qtype == "IfcQuantityCount":
                    print(f"  {quantity.Name}: {quantity.CountValue} (count)")
                elif qtype == "IfcQuantityWeight":
                    print(f"  {quantity.Name}: {quantity.WeightValue} (weight)")
```

---

## Example 8: Schema Version Handling

```python
# IfcOpenShell 0.8.x — handle different IFC schema versions
import ifcopenshell

model = ifcopenshell.open("model.ifc")
schema = model.schema

if schema == "IFC2X3":
    # IFC2X3 uses IfcWallStandardCase (removed in IFC4)
    walls = model.by_type("IfcWallStandardCase") + model.by_type("IfcWall")
    # Spatial structure uses IfcSpatialStructureElement
    storeys = model.by_type("IfcBuildingStorey")

elif schema == "IFC4":
    # IFC4 uses IfcWall with PredefinedType
    walls = model.by_type("IfcWall")
    storeys = model.by_type("IfcBuildingStorey")

elif schema == "IFC4X3":
    # IFC4X3 adds infrastructure entities
    walls = model.by_type("IfcWall")
    storeys = model.by_type("IfcBuildingStorey")
    # Infrastructure-specific queries (ONLY available in IFC4X3)
    alignments = model.by_type("IfcAlignment")
    roads = model.by_type("IfcRoad")
    bridges = model.by_type("IfcBridge")
    facilities = model.by_type("IfcFacility")
```

```typescript
// web-ifc — schema version check
const schema = ifcApi.GetModelSchema(modelID);

if (schema === "IFC4X3") {
  // Check if tool supports infrastructure entities
  console.warn("IFC4X3 detected — verify geometry support for alignment entities");
}
```

---

## Example 9: Cross-Tool Data Transfer Pattern

```python
# Pattern: Extract IFC data for use in web-ifc / Three.js viewer
# This shows what data to extract and how to structure it for the target

import ifcopenshell
import ifcopenshell.util.element
import ifcopenshell.util.unit
import ifcopenshell.geom
import json

model = ifcopenshell.open("model.ifc")
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

export_data = {
    "schema": model.schema,
    "unitScale": unit_scale,
    "elements": []
}

settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

for wall in model.by_type("IfcWall"):
    element_data = {
        "expressID": wall.id(),
        "globalId": wall.GlobalId,
        "name": wall.Name,
        "ifcClass": wall.is_a(),
        "properties": ifcopenshell.util.element.get_psets(wall),
        "container": None,
        "type": None,
    }

    # Spatial container
    container = ifcopenshell.util.element.get_container(wall)
    if container:
        element_data["container"] = {
            "expressID": container.id(),
            "name": container.Name,
            "ifcClass": container.is_a()
        }

    # Type object
    wall_type = ifcopenshell.util.element.get_type(wall)
    if wall_type:
        element_data["type"] = {
            "expressID": wall_type.id(),
            "name": wall_type.Name,
            "ifcClass": wall_type.is_a()
        }

    export_data["elements"].append(element_data)

# This JSON can be consumed by a web viewer alongside the IFC file
# The viewer uses web-ifc for geometry and this JSON for pre-extracted semantic data
with open("metadata.json", "w") as f:
    json.dump(export_data, f, indent=2, default=str)
```

---

## Example 10: Speckle IFC Data Structure

```python
# How IFC data appears in Speckle after conversion
# (read-only reference — Speckle handles conversion internally)

# A wall in Speckle looks like this:
speckle_wall = {
    "speckle_type": "Objects.BuiltElements.Wall",
    "applicationId": "2XQ$n5SLP5MBLyL442paFx",  # IFC GlobalId
    "name": "Basic Wall:Generic - 200mm",
    "properties": {
        "IFC Type": "IfcWall",
        "IFC GUID": "2XQ$n5SLP5MBLyL442paFx",
        "IFC Attributes": {
            "Name": "Basic Wall:Generic - 200mm",
            "Description": None,
            "ObjectType": "Generic - 200mm",
            "Tag": "123456"
        },
        "Property Sets": {
            "Pset_WallCommon": {
                "IsExternal": True,
                "LoadBearing": False,
                "FireRating": "REI60"
            },
            "Qto_WallBaseQuantities": {
                "Length": 5.0,
                "Height": 3.0,
                "GrossVolume": 3.0
            }
        }
    },
    "displayValue": [
        # Speckle Mesh objects (tessellated geometry)
        {
            "speckle_type": "Objects.Geometry.Mesh",
            "vertices": [0.0, 0.0, 0.0, 5.0, 0.0, 0.0, ...],
            "faces": [0, 1, 2, 0, 2, 3, ...],
        }
    ],
    "renderMaterial": {
        "speckle_type": "Objects.Other.RenderMaterial",
        "name": "Concrete",
        "diffuse": 4289374890  # ARGB color int
    }
}

# Key insight: Speckle preserves ALL semantic data in the flat properties dict.
# Geometry is ALWAYS tessellated (displayValue).
# Relationships are partially preserved via collection hierarchy.
```
