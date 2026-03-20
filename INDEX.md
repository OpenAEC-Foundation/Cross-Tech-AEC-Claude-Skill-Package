# Cross-Tech AEC Skill Package — Skill Index

> ~15 deterministic skills for cross-technology AEC integration with Claude.
> Each skill bridges exactly ONE technology boundary. English-only. ALWAYS/NEVER deterministic language.

---

## Skills by Category

### Core (2 skills) — Foundation knowledge

These skills define the fundamental concepts every boundary skill builds on: IFC schema mapping and coordinate reference systems.

| # | Skill | Path | Boundary | Description |
|---|-------|------|----------|-------------|
| 1 | crosstech-core-ifc-schema-bridge | `core/` | IFC ↔ All formats | IFC4/IFC4X3 schema structure, entity hierarchy, property sets, and how IFC concepts map to representations in web-ifc, Blender/Bonsai, FreeCAD, Speckle, and Three.js. The universal translation layer for all IFC-related boundary skills. |
| 2 | crosstech-core-coordinate-systems | `core/` | BIM local ↔ GIS global | Coordinate Reference Systems (CRS), EPSG codes, IFC map conversion (IfcMapConversion, IfcProjectedCRS), true north vs project north, local BIM coordinates to global GIS coordinates transformation, and the PROJ/pyproj pipeline for CRS conversions. |

---

### Implementation (10 skills) — Technology boundary bridges

Each implementation skill provides complete, working patterns for crossing one specific technology boundary.

| # | Skill | Path | Boundary | Description |
|---|-------|------|----------|-------------|
| 3 | crosstech-impl-ifc-to-webifc | `impl/` | IfcOpenShell ↔ web-ifc | Converting IFC files between IfcOpenShell (Python, server-side) and web-ifc (WebAssembly, browser-side). Schema compatibility, property access patterns, geometry processing differences, and streaming large models. |
| 4 | crosstech-impl-ifc-to-threejs | `impl/` | IFC ↔ Three.js | Loading IFC models into Three.js scenes via web-ifc-three or @thatopen/components. Mesh conversion, material mapping, selection/picking, property panels, and maintaining IFC metadata in the Three.js scene graph. |
| 5 | crosstech-impl-speckle-blender | `impl/` | Speckle ↔ Blender | Speckle connector for Blender: sending/receiving geometry, object conversion (Speckle Base ↔ Blender mesh/curve/NURBS), material mapping, collection structure, and handling Bonsai/IFC data through Speckle. |
| 6 | crosstech-impl-speckle-revit | `impl/` | Speckle ↔ Revit | Speckle connector for Revit: sending/receiving BIM elements, family type mapping, parameter synchronization, view-based filtering, and round-trip data integrity between Revit families and Speckle objects. |
| 7 | crosstech-impl-qgis-bim-georef | `impl/` | QGIS ↔ BIM/IFC | Georeferencing BIM models in QGIS: importing IFC site coordinates, CRS transformation via IfcMapConversion, overlaying BIM footprints on GIS maps, and exporting georeferenced data back to IFC. |
| 8 | crosstech-impl-ifc-erpnext-costing | `impl/` | IFC ↔ ERPNext | Extracting quantities from IFC models (IfcQuantityArea, IfcQuantityVolume, IfcQuantityLength) and mapping them to ERPNext BOM items, cost estimates, and project budgets. Property set to doctype field mapping. |
| 9 | crosstech-impl-bim-web-viewer | `impl/` | BIM/IFC ↔ Web browser | End-to-end pipeline for BIM web viewing: IFC → web-ifc → Three.js viewer with navigation, selection, property panel, section planes, and annotations. Covers @thatopen/components and custom viewer approaches. |
| 10 | crosstech-impl-freecad-ifc-bridge | `impl/` | FreeCAD ↔ IFC | FreeCAD native IFC support: importing/exporting IFC via FreeCAD's BIM workbench, NativeIFC mode, IfcOpenShell integration within FreeCAD, and round-trip editing of IFC files in FreeCAD. |
| 11 | crosstech-impl-n8n-aec-pipeline | `impl/` | n8n ↔ AEC tools | Automating AEC workflows with n8n: IFC processing triggers, Speckle webhook integration, model validation pipelines, report generation, and connecting AEC tools via REST APIs and custom n8n nodes. |
| 12 | crosstech-impl-docker-aec-stack | `impl/` | Docker ↔ AEC services | Containerizing AEC tool stacks: IfcOpenShell in Docker, Speckle server deployment, QGIS server, web-ifc processing workers, multi-service docker-compose for AEC development environments. |

---

### Errors (2 skills) — Cross-technology error diagnosis

These skills provide diagnostic approaches for when data crossing technology boundaries fails.

| # | Skill | Path | Boundary | Description |
|---|-------|------|----------|-------------|
| 13 | crosstech-errors-conversion | `errors/` | Any format ↔ Any format | Symptom-to-root-cause decision tree for data conversion errors: missing geometry, lost properties, broken relationships, schema version mismatches, encoding issues, and partial conversion failures across all AEC technology boundaries. |
| 14 | crosstech-errors-coordinate-mismatch | `errors/` | BIM local ↔ GIS global | Diagnosing coordinate-related errors: models appearing at wrong location, scale mismatches (mm vs m vs ft), flipped axes (Y-up vs Z-up), missing georeferencing, CRS mismatches, and datum transformation errors. |

---

### Agents (1 skill) — AEC pipeline orchestration

| # | Skill | Path | Boundary | Description |
|---|-------|------|----------|-------------|
| 15 | crosstech-agents-aec-orchestrator | `agents/` | Multi-hop | Central orchestrator for multi-technology AEC pipelines. Selects the right boundary skill(s) for a user request, chains skills for multi-hop scenarios (e.g., Revit → Speckle → Blender → IFC → Web viewer), validates data integrity at each hop, and handles error recovery. |

---

## Totals

| Category | Skills | Status |
|----------|--------|--------|
| Core | 2 | NOT STARTED |
| Implementation | 10 | NOT STARTED |
| Errors | 2 | NOT STARTED |
| Agents | 1 | NOT STARTED |
| **Total** | **~15** | **Preliminary** |

---

## Technology Boundary Matrix

```
                 Blender  IfcOpenShell  Speckle  QGIS  Three.js  web-ifc  FreeCAD  ERPNext  n8n  Docker
Blender            —         (pkg)       #5       —      —         —        —        —       —     —
IfcOpenShell      (pkg)       —          —        —      —         #3       —        #8      —     #12
Speckle            #5         —          —        —      —         —        —        —       #11   #12
QGIS               —         —          —        —      —         —        —        —       —     #12
Three.js           —         —          —        —      —         #4       —        —       —     —
web-ifc            —         #3         —        —      #4        —        —        —       —     #12
FreeCAD            —         #10        —        —      —         —        —        —       —     —
ERPNext            —         #8         —        —      —         —        —        —       #11   #12
n8n                —         —          #11      —      —         —        —        #11     —     #12
Docker             —         #12        #12      #12    —         #12      —        #12     #12   —

(pkg) = covered by the dependent single-technology package, not this package
#N = skill number in this package
```

---

## Skill Dependencies

```
agents/aec-orchestrator  ──> ALL impl/* skills (routes by technology boundary)
agents/aec-orchestrator  ──> errors/* skills (error recovery)

impl/* skills            ──> core/ifc-schema-bridge (IFC entity mapping)
impl/* skills            ──> core/coordinate-systems (CRS transformations)

errors/conversion        ──> core/ifc-schema-bridge (schema validation)
errors/coordinate-mismatch ──> core/coordinate-systems (CRS diagnostics)
```

---

**Version:** 0.1 (preliminary) — All skills NOT STARTED.
