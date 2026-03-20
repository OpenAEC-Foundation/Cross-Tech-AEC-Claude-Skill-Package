# crosstech-agents-aec-orchestrator — Methods Reference

## Skill Routing Logic

### Route Selection Algorithm

```
FUNCTION select_skills(user_request):
    technologies = extract_technologies(user_request)
    boundaries = identify_boundaries(technologies)

    skills_to_load = []

    FOR EACH boundary IN boundaries:
        skill = ROUTING_TABLE.lookup(boundary)
        IF skill.status == "AVAILABLE":
            skills_to_load.append(skill)
        ELSE:
            WARN user: "{skill.name} is DEFERRED — requires {skill.required_package}"
            IF skill.workaround EXISTS:
                skills_to_load.append(skill.workaround)

    # ALWAYS add foundation skills when relevant
    IF any_hop_involves_ifc(boundaries):
        skills_to_load.prepend("crosstech-core-ifc-schema-bridge")
    IF any_hop_involves_coordinates(boundaries):
        skills_to_load.prepend("crosstech-core-coordinate-systems")

    RETURN deduplicate(skills_to_load)
```

### Technology Detection Keywords

| Technology | Detection Keywords |
|------------|-------------------|
| IFC / IfcOpenShell | ifc, ifcopenshell, bim model, ifc4, ifc4x3, property set, quantity set |
| web-ifc / ThatOpen | web-ifc, webifc, thatopen, @thatopen/components, ifc.js, browser ifc |
| Three.js | three.js, threejs, 3d viewer, webgl, scene graph, mesh viewer |
| Blender / Bonsai | blender, bonsai, bpy, blend file, 3d modeling |
| Speckle | speckle, speckle connector, speckle server, data transport |
| Revit | revit, revit api, autodesk, family, revit parameter |
| QGIS | qgis, gis, geospatial, shapefile, geopackage, georeferencing |
| FreeCAD | freecad, freecad bim, nativeifc, freecad ifc |
| ERPNext | erpnext, erp, bom, bill of materials, cost estimate, frappe |
| n8n | n8n, workflow automation, webhook, pipeline trigger |
| Docker | docker, container, docker-compose, dockerfile, containerize |

### Boundary Identification Matrix

Given two detected technologies, look up the boundary skill:

| From \ To | IFC | web-ifc | Three.js | Blender | Speckle | Revit | QGIS | FreeCAD | ERPNext | n8n | Docker |
|-----------|-----|---------|----------|---------|---------|-------|------|---------|---------|-----|--------|
| **IFC** | — | #10 (D) | #11 (D) | (pkg) | (pkg) | (pkg) | #14 (D) | #4 | #3 | #5 | #6 |
| **web-ifc** | #10 (D) | — | #15 (D) | — | — | — | — | — | — | — | #6 |
| **Three.js** | #11 (D) | #15 (D) | — | — | — | — | — | — | — | — | — |
| **Blender** | (pkg) | — | — | — | #12 (D) | — | — | — | — | — | — |
| **Speckle** | (pkg) | — | — | #12 (D) | — | #13 (D) | — | — | — | #5 | #6 |
| **Revit** | (pkg) | — | — | — | #13 (D) | — | — | — | — | — | — |
| **QGIS** | #14 (D) | — | — | — | — | — | — | — | — | — | #6 |
| **FreeCAD** | #4 | — | — | — | — | — | — | — | — | — | — |
| **ERPNext** | #3 | — | — | — | — | — | — | — | — | #5 | #6 |
| **n8n** | #5 | — | — | — | #5 | — | — | — | #5 | — | #6 |
| **Docker** | #6 | #6 | — | — | #6 | — | #6 | — | #6 | #6 | — |

Legend: `(D)` = DEFERRED (Wave B), `(pkg)` = covered by dependent single-technology package, `#N` = skill number

---

## Validation Methods

### validate_hop()

**Purpose**: Verify data integrity after a boundary crossing.

**Signature**:
```python
def validate_hop(
    input_data: Any,
    output_data: Any,
    hop_name: str,
    checks: list[str] = None
) -> ValidationResult:
```

**Parameters**:
- `input_data` — Data BEFORE the hop (IFC model, JSON, file path)
- `output_data` — Data AFTER the hop
- `hop_name` — Human-readable hop description for error messages
- `checks` — Optional subset of checks to run (default: ALL)

**Available checks**:
| Check Name | Applies To | What It Verifies |
|-----------|-----------|-----------------|
| `schema_version` | IFC ↔ IFC | Schema identifier unchanged |
| `entity_count` | IFC ↔ IFC | No silent entity drops |
| `property_spot_check` | IFC ↔ Any | Random sample of 3 elements retains properties |
| `geometry_presence` | IFC ↔ Any | Elements with geometry retain geometry |
| `unit_consistency` | Any ↔ Any | Output units match expected target |
| `coordinate_range` | Any ↔ GIS | Coordinates within valid CRS bounds |
| `spatial_hierarchy` | IFC ↔ Any | Site → Building → Storey structure preserved |
| `file_validity` | Any ↔ File | Output file non-zero and parseable |

**Return**: `ValidationResult` with `.passed: bool`, `.errors: list[str]`, `.warnings: list[str]`

### validate_pipeline()

**Purpose**: Run validation across an entire multi-hop pipeline.

**Signature**:
```python
def validate_pipeline(
    hops: list[HopDefinition],
    intermediate_results: list[Any]
) -> PipelineValidationResult:
```

**Logic**:
1. For each consecutive pair `(result[i], result[i+1])`, run `validate_hop()`
2. Aggregate all errors and warnings
3. If ANY hop has critical errors, mark the pipeline as FAILED
4. Return the first failing hop index for targeted debugging

---

## Pipeline Composition Methods

### compose_pipeline()

**Purpose**: Build a multi-hop pipeline from a user request.

**Algorithm**:
```
FUNCTION compose_pipeline(user_request):
    1. Extract source technology and target technology
    2. Find shortest path through the boundary matrix
    3. For each edge in the path:
       a. Look up the boundary skill
       b. If DEFERRED: check for workaround or STOP
       c. If AVAILABLE: add to pipeline
    4. Insert core skills at the beginning:
       - ifc-schema-bridge if ANY hop involves IFC
       - coordinate-systems if ANY hop involves coordinates
    5. Insert validation checkpoints between each hop
    6. Return ordered list of (skill, validation_checks) tuples
```

### find_shortest_path()

**Purpose**: Find the minimum-hop route between two technologies.

**Method**: Breadth-first search on the boundary identification matrix. Each cell with a skill (available or deferred) is a valid edge.

**Constraints**:
- Maximum path length: 4 hops
- ALWAYS prefer paths with all AVAILABLE skills over paths with DEFERRED skills
- If no path exists with only AVAILABLE skills, show the shortest path and mark DEFERRED hops

---

## Error Routing Methods

### route_error()

**Purpose**: Given a failure at a specific hop, load the correct error diagnosis skill.

**Decision table**:

| Error Category | Symptoms | Skill to Load |
|---------------|----------|---------------|
| Conversion failure | Missing geometry, lost properties, broken relationships, schema mismatch | `crosstech-errors-conversion` |
| Coordinate failure | Wrong location, scale mismatch, axis flip, CRS error | `crosstech-errors-coordinate-mismatch` |
| Tool API error | HTTP 4xx/5xx, timeout, authentication failure | Reload the `impl/` skill for that boundary |
| Pipeline infrastructure | Docker container crash, n8n workflow timeout, file I/O | `crosstech-impl-docker-aec-stack` or `crosstech-impl-n8n-aec-pipeline` |

### escalate_to_human()

**Purpose**: After 3 failed retries, stop automated recovery and present diagnostics.

**Output**:
1. Which hop failed (number and boundary)
2. Error messages from all 3 attempts
3. Which error skill was consulted
4. Suggested manual investigation steps
