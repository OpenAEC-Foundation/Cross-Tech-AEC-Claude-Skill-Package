# crosstech-core-ifc-schema-bridge: IFC Entity Mapping Tables

Sources:
- https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/
- https://ifc43-docs.standards.buildingsmart.org/
- https://docs.ifcopenshell.org/
- https://github.com/ThatOpen/engine_web-ifc
- https://docs.speckle.systems/developers/data-schema/connectors/ifc-schema

---

## 1. IFC Entity Hierarchy — Complete Reference

### Abstract Root Types

| Entity | Parent | Purpose | Has GlobalId |
|--------|--------|---------|:------------:|
| `IfcRoot` | — | Abstract base for all identifiable entities | Yes |
| `IfcObjectDefinition` | `IfcRoot` | Abstract base for all objects and types | Yes |
| `IfcObject` | `IfcObjectDefinition` | Abstract base for individual occurrences | Yes |
| `IfcProduct` | `IfcObject` | Abstract base for physical/spatial objects | Yes |
| `IfcElement` | `IfcProduct` | Abstract base for building elements | Yes |
| `IfcRelationship` | `IfcRoot` | Abstract base for objectified relationships | Yes |
| `IfcPropertyDefinition` | `IfcRoot` | Abstract base for property containers | Yes |

### Concrete Building Element Types (IFC4 + IFC4X3)

| IFC Entity | PredefinedType Values | Category |
|-----------|----------------------|----------|
| `IfcWall` | MOVABLE, PARAPET, PARTITIONING, PLUMBINGWALL, SHEAR, SOLIDWALL, STANDARD, POLYGONAL, ELEMENTEDWALL | Structural |
| `IfcSlab` | FLOOR, ROOF, LANDING, BASESLAB, APPROACH_SLAB, PAVING, WEARING | Structural |
| `IfcBeam` | BEAM, JOIST, HOLLOWCORE, LINTEL, SPANDREL, T_BEAM | Structural |
| `IfcColumn` | COLUMN, PILASTER, STANDCOLUMN | Structural |
| `IfcDoor` | DOOR, GATE, TRAPDOOR | Opening |
| `IfcWindow` | WINDOW, SKYLIGHT, LIGHTDOME | Opening |
| `IfcStair` | STRAIGHT_RUN, TWO_CURVED_RUN, QUARTER_WINDING, SPIRAL, ... | Circulation |
| `IfcRamp` | STRAIGHT_RUN, TWO_STRAIGHT_RUN, QUARTER_TURN, SPIRAL, ... | Circulation |
| `IfcRoof` | FLAT_ROOF, SHED_ROOF, GABLE_ROOF, HIP_ROOF, ... | Envelope |
| `IfcCurtainWall` | USERDEFINED, NOTDEFINED | Envelope |
| `IfcPlate` | CURTAIN_PANEL, SHEET, FLANGE_PLATE, WEB_PLATE | Envelope |
| `IfcMember` | BRACE, CHORD, COLLAR, MEMBER, MULLION, ... | Structural |
| `IfcFooting` | CAISSON_FOUNDATION, FOOTING_BEAM, PAD_FOOTING, PILE_CAP, STRIP_FOOTING | Foundation |
| `IfcPile` | BORED, DRIVEN, JETGROUTING, COHESION, FRICTION, SUPPORT | Foundation |
| `IfcOpeningElement` | OPENING, RECESS | Void |
| `IfcFurnishingElement` | USERDEFINED, NOTDEFINED | Furniture |

### Concrete Infrastructure Types (IFC4X3 ONLY)

| IFC Entity | Parent | Purpose |
|-----------|--------|---------|
| `IfcAlignment` | `IfcLinearPositioningElement` | Road/railway alignment axis |
| `IfcAlignmentHorizontal` | `IfcAlignment` | Horizontal alignment geometry |
| `IfcAlignmentVertical` | `IfcAlignment` | Vertical profile |
| `IfcAlignmentCant` | `IfcAlignment` | Superelevation |
| `IfcRoad` | `IfcFacility` | Road facility |
| `IfcRailway` | `IfcFacility` | Railway facility |
| `IfcBridge` | `IfcFacility` | Bridge structure |
| `IfcMarineFacility` | `IfcFacility` | Port/dock/harbor |
| `IfcFacilityPart` | `IfcSpatialElement` | Generalized spatial part |
| `IfcCourse` | `IfcBuiltElement` | Road course layer |
| `IfcPavement` | `IfcBuiltElement` | Pavement structure |
| `IfcKerb` | `IfcBuiltElement` | Kerb/curb element |
| `IfcEarthworksCut` | `IfcBuiltElement` | Excavation |
| `IfcEarthworksFill` | `IfcBuiltElement` | Fill placement |
| `IfcBorehole` | `IfcGeoScienceElement` | Geotechnical bore |
| `IfcGeomodel` | `IfcGeoScienceElement` | 3D geological model |
| `IfcGeoslice` | `IfcGeoScienceElement` | Geological slice |
| `IfcSolidStratum` | `IfcGeoScienceElement` | Geological layer |

### Spatial Structure Types

| Entity | IFC4 | IFC4X3 | Typical Contains |
|--------|:----:|:------:|-----------------|
| `IfcProject` | Yes | Yes | Top-level container (ALWAYS exactly one) |
| `IfcSite` | Yes | Yes | Buildings, facilities |
| `IfcBuilding` | Yes | Yes (subtype of `IfcFacility`) | Building storeys |
| `IfcBuildingStorey` | Yes | Yes | Building elements |
| `IfcSpace` | Yes | Yes | Space boundaries |
| `IfcFacility` | No | Yes | Generalized container (roads, bridges) |
| `IfcFacilityPart` | No | Yes | Infrastructure parts |

### Type Object Types

Each element type has a corresponding `IfcElementType` that holds shared properties and representations:

| Occurrence | Type |
|-----------|------|
| `IfcWall` | `IfcWallType` |
| `IfcSlab` | `IfcSlabType` |
| `IfcBeam` | `IfcBeamType` |
| `IfcColumn` | `IfcColumnType` |
| `IfcDoor` | `IfcDoorType` |
| `IfcWindow` | `IfcWindowType` |
| `IfcMember` | `IfcMemberType` |
| `IfcPlate` | `IfcPlateType` |
| `IfcCurtainWall` | `IfcCurtainWallType` |
| `IfcStair` | `IfcStairType` |
| `IfcRamp` | `IfcRampType` |
| `IfcRoof` | `IfcRoofType` |

---

## 2. Relationship Types — Complete Reference

### Core Relationships

| Relationship | Relating Side | Related Side | Cardinality | Purpose |
|-------------|--------------|-------------|:-----------:|---------|
| `IfcRelAggregates` | `RelatingObject` (1) | `RelatedObjects` (1..n) | 1:N | Spatial decomposition (Project→Site→Building→Storey) |
| `IfcRelContainedInSpatialStructure` | `RelatingStructure` (1) | `RelatedElements` (1..n) | 1:N | Element containment (Storey→Walls) |
| `IfcRelDefinesByProperties` | `RelatingPropertyDefinition` (1) | `RelatedObjects` (1..n) | 1:N | Property set attachment |
| `IfcRelDefinesByType` | `RelatingType` (1) | `RelatedObjects` (1..n) | 1:N | Type assignment |
| `IfcRelAssociatesMaterial` | `RelatingMaterial` (1) | `RelatedObjects` (1..n) | 1:N | Material assignment |
| `IfcRelVoidsElement` | `RelatingBuildingElement` (1) | `RelatedOpeningElement` (1) | 1:1 | Void cutting |
| `IfcRelFillsElement` | `RelatingOpeningElement` (1) | `RelatedBuildingElement` (1) | 1:1 | Fill opening (door in opening) |
| `IfcRelSpaceBoundary` | `RelatingSpace` (1) | `RelatedBuildingElement` (1) | 1:1 | Thermal boundary |
| `IfcRelAssignsToGroup` | `RelatingGroup` (1) | `RelatedObjects` (1..n) | 1:N | Group membership |
| `IfcRelDeclares` | `RelatingContext` (1) | `RelatedDefinitions` (1..n) | 1:N | Declaration in project |
| `IfcRelPositions` (IFC4X3) | `RelatingPositioningElement` (1) | `RelatedProducts` (1..n) | 1:N | Linear positioning |

---

## 3. Property Set Structure

### IfcPropertySet

```
IfcPropertySet
├── Name: "Pset_WallCommon"
├── HasProperties: [
│   ├── IfcPropertySingleValue
│   │   ├── Name: "IsExternal"
│   │   ├── NominalValue: IfcBoolean(True)
│   │   └── Unit: (none)
│   ├── IfcPropertySingleValue
│   │   ├── Name: "LoadBearing"
│   │   ├── NominalValue: IfcBoolean(False)
│   │   └── Unit: (none)
│   └── IfcPropertyEnumeratedValue
│       ├── Name: "FireRating"
│       └── EnumerationValues: [IfcLabel("REI60")]
│   ]
└── (attached to elements via IfcRelDefinesByProperties)
```

### Standard Property Set Prefixes

| Prefix | Purpose | Example |
|--------|---------|---------|
| `Pset_` | Standard property sets (buildingSMART) | `Pset_WallCommon`, `Pset_DoorCommon` |
| `Qto_` | Standard quantity sets | `Qto_WallBaseQuantities` |
| (custom) | Project-specific property sets | `MyProject_StructuralData` |

### IfcPropertySingleValue — Value Types

| IFC Value Type | Python Type (IfcOpenShell) | JS Type (web-ifc) |
|---------------|--------------------------|-------------------|
| `IfcLabel` | `str` | `{ type: 1, value: "string" }` |
| `IfcText` | `str` | `{ type: 1, value: "string" }` |
| `IfcReal` | `float` | `{ type: 4, value: number }` |
| `IfcInteger` | `int` | `{ type: 3, value: number }` |
| `IfcBoolean` | `bool` | `{ type: 3, value: 0 or 1 }` |
| `IfcLogical` | `str` ("TRUE"/"FALSE"/"UNKNOWN") | `{ type: 3, value: number }` |
| `IfcIdentifier` | `str` | `{ type: 1, value: "string" }` |
| `IfcLengthMeasure` | `float` | `{ type: 4, value: number }` |
| `IfcAreaMeasure` | `float` | `{ type: 4, value: number }` |
| `IfcVolumeMeasure` | `float` | `{ type: 4, value: number }` |

---

## 4. Quantity Set Structure

### IfcElementQuantity

```
IfcElementQuantity
├── Name: "Qto_WallBaseQuantities"
├── Quantities: [
│   ├── IfcQuantityLength
│   │   ├── Name: "Length"
│   │   └── LengthValue: 5.0
│   ├── IfcQuantityArea
│   │   ├── Name: "GrossSideArea"
│   │   └── AreaValue: 15.0
│   ├── IfcQuantityVolume
│   │   ├── Name: "GrossVolume"
│   │   └── VolumeValue: 3.0
│   └── IfcQuantityWeight
│       ├── Name: "GrossWeight"
│       └── WeightValue: 7200.0
│   ]
└── (attached via IfcRelDefinesByProperties — same as property sets)
```

### Quantity Types

| IFC Type | Value Attribute | Typical Unit |
|----------|----------------|-------------|
| `IfcQuantityLength` | `LengthValue` | meters |
| `IfcQuantityArea` | `AreaValue` | square meters |
| `IfcQuantityVolume` | `VolumeValue` | cubic meters |
| `IfcQuantityCount` | `CountValue` | dimensionless |
| `IfcQuantityWeight` | `WeightValue` | kilograms |
| `IfcQuantityTime` | `TimeValue` | seconds |

---

## 5. Per-Tool Entity Mapping — Detailed

### IFC → IfcOpenShell

| IFC Concept | IfcOpenShell API | Access Pattern |
|------------|-----------------|----------------|
| Any entity by type | `model.by_type("IfcWall")` | Returns `list[entity_instance]` |
| Entity by ID | `model.by_id(122)` | Returns single `entity_instance` |
| Entity by GUID | `model.by_guid("2XQ$n5...")` | Returns single `entity_instance` |
| Entity class | `entity.is_a()` or `entity.is_a("IfcProduct")` | String or bool |
| All attributes | `entity.get_info()` or `entity.get_info(recursive=True)` | Dict |
| Property sets | `ifcopenshell.util.element.get_psets(entity)` | Nested dict |
| Type object | `ifcopenshell.util.element.get_type(entity)` | `entity_instance` or `None` |
| Spatial container | `ifcopenshell.util.element.get_container(entity)` | `entity_instance` or `None` |
| Children | `ifcopenshell.util.element.get_decomposition(entity)` | List |
| Geometry | `ifcopenshell.geom.create_shape(settings, entity)` | Tessellated mesh |
| Placement matrix | `ifcopenshell.util.placement.get_local_placement(entity.ObjectPlacement)` | 4x4 numpy array |
| Unit scale | `ifcopenshell.util.unit.calculate_unit_scale(model)` | Float (to SI meters) |
| Inverse refs | `model.get_inverse(entity)` | Set of entity_instances |
| Schema version | `model.schema` | `"IFC2X3"`, `"IFC4"`, `"IFC4X3"` |

### IFC → web-ifc

| IFC Concept | web-ifc API | Access Pattern |
|------------|------------|----------------|
| Any entity by ID | `ifcApi.GetLine(modelID, expressID, true)` | `IfcLineObject` |
| Entity type code | `ifcApi.GetLineType(modelID, expressID)` | Numeric constant (e.g., `IFCWALL`) |
| All entities | `ifcApi.GetAllLines(modelID)` | Array of expressIDs |
| Entity attribute | `line.Name.value`, `line.GlobalId.value` | Wrapped values |
| Property sets | Manual: traverse `IfcRelDefinesByProperties` | No helper function |
| Geometry | `ifcApi.GetFlatMesh(modelID, expressID)` | `FlatMesh` with vertex/index arrays |
| All geometry | `ifcApi.StreamAllMeshes(modelID, callback)` | Streaming callback |
| Coordination matrix | `ifcApi.GetCoordinationMatrix(modelID)` | `number[]` (4x4 flat) |
| Schema version | `ifcApi.GetModelSchema(modelID)` | String |
| Create entity | `ifcApi.WriteLine(modelID, lineObject)` | Returns expressID |
| Save | `ifcApi.SaveModel(modelID)` | `Uint8Array` |

### IFC → Blender + Bonsai

| IFC Concept | Blender Representation | Bonsai Requirement |
|------------|----------------------|-------------------|
| `IfcProduct` | `bpy.types.Object` | Bonsai maintains IFC link |
| `IfcProduct.Name` | `object.name` | Synced by Bonsai |
| Geometry | `bpy.types.Mesh` (tessellated) | Original IFC geometry preserved internally |
| `IfcPropertySet` | Custom properties via Bonsai | REQUIRES Bonsai — raw import loses properties |
| `IfcBuildingStorey` | Collection | Spatial hierarchy as collection tree |
| `IfcRelAggregates` | Collection parent-child | Partial mapping |
| `IfcMaterial` | `bpy.types.Material` (simplified) | Layer sets simplified to single material |
| `IfcRelDefinesByType` | Shared data block | Managed by Bonsai |

### IFC → Three.js (via ThatOpen Components)

| IFC Concept | Three.js Representation | Notes |
|------------|------------------------|-------|
| `IfcProduct` | `THREE.Mesh` or `THREE.Object3D` | One mesh per element or batched into Fragments |
| Geometry | `THREE.BufferGeometry` | Tessellated triangles |
| Spatial hierarchy | `Object3D` parent-child | Partial — often flattened for GPU performance |
| Properties | NOT in scene graph | MUST query via web-ifc `GetLine()` separately |
| Relationships | NOT in scene graph | MUST maintain parallel `Map<expressID, data>` |
| Materials | `THREE.MeshLambertMaterial` | Basic color only — IFC material layers lost |
| expressID | Stored in geometry attributes | Used to link back to web-ifc data |

### IFC → Speckle

| IFC Concept | Speckle Representation | Preserved? |
|------------|----------------------|:----------:|
| IFC entity class | `properties["IFC Type"]` | Yes |
| GlobalId | `applicationId` + `properties["IFC GUID"]` | Yes |
| `IfcPropertySet` | `properties["Property Sets"][psetName]` | Yes |
| IFC attributes | `properties["IFC Attributes"]` | Yes |
| Geometry | `displayValue[]` as Speckle Mesh | Tessellated |
| Spatial hierarchy | Collection tree | Yes |
| Materials | `RenderMaterial` proxy | Simplified |
| Relationships | Partial via hierarchy | Some lost |

### IFC → FreeCAD

| IFC Concept | NativeIFC Mode | Legacy Mode |
|------------|---------------|-------------|
| Entity access | Direct IFC (no mapping) | `Part::Feature` / Arch object |
| Geometry | IFC geometry (parametric preserved) | OpenCASCADE BREP (BREP preserved, others tessellated) |
| Properties | Full IFC access | Partial (depends on import settings) |
| Relationships | Full IFC access | Simplified / lost |
| Round-trip | Lossless | Lossy |

---

## 6. Geometry Representation Types

IFC supports multiple geometry representations. EVERY target tool tessellates for display.

| IFC Geometry Type | Description | Round-trip Safe? |
|------------------|-------------|:----------------:|
| `IfcExtrudedAreaSolid` | Profile + direction + depth | Only in IFC-native tools |
| `IfcSweptDiskSolid` | Circular sweep along path | Only in IFC-native tools |
| `IfcBooleanClippingResult` | CSG boolean operations | Only in IFC-native tools |
| `IfcFacetedBrep` | BREP with flat faces | Partially (FreeCAD preserves BREP) |
| `IfcTriangulatedFaceSet` | Pre-tessellated triangles | Yes (already tessellated) |
| `IfcPolygonalFaceSet` | Pre-tessellated polygons | Yes (already tessellated) |
| `IfcMappedItem` | Reuse of shared geometry | Lost in most targets (each instance gets own copy) |

**Tessellation impact**: Converting `IfcExtrudedAreaSolid` (a few parameters) to triangulated mesh produces 10-100x more data. NEVER replace parametric geometry with tessellated versions in the IFC file.
