# crosstech-errors-conversion — Diagnostic Methods

## IfcOpenShell Diagnostic Commands

### Check Schema Version
```python
import ifcopenshell
model = ifcopenshell.open("model.ifc")
print(f"Schema: {model.schema}")  # "IFC2X3", "IFC4", "IFC4X3"
```

### Check Unit Scale
```python
import ifcopenshell.util.unit
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
print(f"Unit scale to meters: {unit_scale}")
# 1.0 = meters, 0.001 = millimeters, 0.3048 = feet
```

### Verify Element Has Geometry
```python
import ifcopenshell.geom
settings = ifcopenshell.geom.settings()
settings.set(settings.USE_WORLD_COORDS, True)

element = model.by_id(express_id)
try:
    shape = ifcopenshell.geom.create_shape(settings, element)
    vert_count = len(shape.geometry.verts) // 3
    face_count = len(shape.geometry.faces) // 3
    print(f"Vertices: {vert_count}, Faces: {face_count}")
except RuntimeError as e:
    print(f"Geometry extraction failed: {e}")
```

### List All Property Sets on an Element
```python
import ifcopenshell.util.element

# Occurrence + type properties combined
psets = ifcopenshell.util.element.get_psets(element)
for pset_name, props in psets.items():
    print(f"\n{pset_name}:")
    for prop_name, value in props.items():
        if prop_name != "id":
            print(f"  {prop_name}: {value}")
```

### List All Quantity Sets on an Element
```python
qtos = ifcopenshell.util.element.get_psets(element, qtos_only=True)
for qto_name, quantities in qtos.items():
    print(f"\n{qto_name}:")
    for q_name, value in quantities.items():
        if q_name != "id":
            print(f"  {q_name}: {value}")
```

### Check Spatial Containment
```python
container = ifcopenshell.util.element.get_container(element)
if container:
    print(f"Contained in: {container.is_a()} '{container.Name}'")
else:
    print("WARNING: Element has no spatial containment")
```

### Check Type Assignment
```python
element_type = ifcopenshell.util.element.get_type(element)
if element_type:
    print(f"Type: {element_type.is_a()} '{element_type.Name}'")
else:
    print("WARNING: Element has no type assignment")
```

### Check Material Assignment
```python
material = ifcopenshell.util.element.get_material(element)
if material:
    print(f"Material: {material.is_a()} '{material.Name if hasattr(material, 'Name') else 'N/A'}'")
else:
    print("WARNING: Element has no material assignment")
```

### Count Elements by Type
```python
from collections import Counter
type_counts = Counter(e.is_a() for e in model.by_type("IfcProduct"))
for ifc_type, count in type_counts.most_common():
    print(f"  {ifc_type}: {count}")
```

### Detect Orphaned Elements (No Spatial Containment)
```python
orphaned = []
for product in model.by_type("IfcProduct"):
    if product.is_a("IfcSpatialElement"):
        continue
    container = ifcopenshell.util.element.get_container(product)
    if not container:
        orphaned.append(product)
print(f"Orphaned elements: {len(orphaned)}")
for e in orphaned[:10]:
    print(f"  #{e.id()} {e.is_a()} '{e.Name}'")
```

### Detect Encoding Issues in Properties
```python
import re
for pset in model.by_type("IfcPropertySingleValue"):
    val = pset.NominalValue
    if val and hasattr(val, 'wrappedValue'):
        s = str(val.wrappedValue)
        if re.search(r'\\X[24]\\', s):
            print(f"Unescaped encoding: #{pset.id()} value='{s}'")
```

---

## web-ifc Diagnostic Commands

### Check Schema Version
```javascript
const schema = ifcApi.GetModelSchema(modelID);
console.log(`Schema: ${schema}`);
```

### Verify Entity Exists
```javascript
const line = ifcApi.GetLine(modelID, expressID, true);
if (!line) {
    console.error(`Entity #${expressID} not found`);
} else {
    console.log(`Type: ${line.constructor.name}`);
}
```

### Get All Entity Types in Model
```javascript
const types = ifcApi.GetAllTypesOfModel(modelID);
for (const type of types) {
    console.log(`${type.typeName}: ${type.typeCount} entities`);
}
```

### Check Geometry Exists
```javascript
try {
    const mesh = ifcApi.GetFlatMesh(modelID, expressID);
    console.log(`Geometries: ${mesh.geometries.size()}`);
    for (let i = 0; i < mesh.geometries.size(); i++) {
        const geom = mesh.geometries.get(i);
        const data = ifcApi.GetGeometry(modelID, geom.geometryExpressID);
        console.log(`  Geometry ${i}: ${data.GetVertexDataSize()} bytes vertex data`);
    }
} catch (e) {
    console.error(`Geometry extraction failed: ${e.message}`);
}
```

### Get Coordination Matrix
```javascript
const matrix = ifcApi.GetCoordinationMatrix(modelID);
console.log("Coordination matrix:", matrix);
// Non-identity matrix indicates georeferencing or offset
```

### Traverse Property Sets (web-ifc)
```javascript
const relIDs = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCRELDEFINESBYPROPERTIES);
for (let i = 0; i < relIDs.size(); i++) {
    const rel = ifcApi.GetLine(modelID, relIDs.get(i), true);
    const related = rel.RelatedObjects.map(o => o.value);
    if (related.includes(targetExpressID)) {
        const pset = ifcApi.GetLine(modelID, rel.RelatingPropertyDefinition.value, true);
        console.log(`PropertySet: ${pset.Name?.value}`);
    }
}
```

---

## Blender / Bonsai Diagnostic Commands

### Check If Bonsai Is Active
```python
import bpy
bonsai_active = hasattr(bpy.context.scene, "BIMProperties")
print(f"Bonsai active: {bonsai_active}")
```

### Check IFC Data Integrity in Bonsai
```python
import blenderbim.tool as tool
ifc_file = tool.Ifc.get()
if ifc_file:
    print(f"Schema: {ifc_file.schema}")
    print(f"Products: {len(ifc_file.by_type('IfcProduct'))}")
else:
    print("No IFC file loaded in Bonsai")
```

### Verify Object Has IFC Link
```python
obj = bpy.context.active_object
ifc_id = obj.BIMObjectProperties.ifc_definition_id if hasattr(obj, 'BIMObjectProperties') else 0
if ifc_id:
    print(f"Linked to IFC entity #{ifc_id}")
else:
    print("WARNING: Object has no IFC link — pure Blender object")
```

---

## Speckle Diagnostic Commands

### Check Received Object Properties
```python
from specklepy.api.client import SpeckleClient
from specklepy.api import operations

# After receiving an object
received = operations.receive(obj_id, transport)
print(f"Type: {received.speckle_type}")
if hasattr(received, 'properties'):
    for key, value in received.properties.items():
        print(f"  {key}: {type(value).__name__}")
```

### Verify IFC Properties Survived Transport
```python
# Check for IFC-specific properties
ifc_type = received.properties.get("IFC Type", "MISSING")
ifc_guid = received.properties.get("IFC GUID", "MISSING")
psets = received.properties.get("Property Sets", {})
print(f"IFC Type: {ifc_type}")
print(f"IFC GUID: {ifc_guid}")
print(f"Property Sets: {len(psets)} found")
```

---

## FreeCAD Diagnostic Commands

### Check NativeIFC Mode
```python
import FreeCAD
doc = FreeCAD.ActiveDocument
for obj in doc.Objects:
    if hasattr(obj, "IfcFilePath"):
        print(f"NativeIFC object: {obj.Name} -> {obj.IfcFilePath}")
```

### Verify Shape After Import
```python
obj = FreeCAD.ActiveDocument.getObject("Wall001")
if obj.Shape.isNull():
    print("WARNING: Shape is null — geometry import failed")
else:
    print(f"Volume: {obj.Shape.Volume}")
    print(f"Faces: {len(obj.Shape.Faces)}")
```
