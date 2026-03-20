# Cross-Tech AEC Skill Package — Definitive Masterplan

## Status
Phase B3 complete. Finalized from raw masterplan after vooronderzoek review.
Date: 2026-03-20

---

## Refinement Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| R-01 | **Two-wave strategy** adopted (D-006) | 4 of 8 dependent packages are skeleton repos. Wave A (9 skills) builds now; Wave B (6 skills) deferred. |
| R-02 | **FreeCAD skill is self-contained** (D-007) | No FreeCAD skill package exists. Must document both sides fully. |
| R-03 | **No scope changes** to preliminary skill list | Research confirms all 15 skills are valid. No merges, splits, or additions needed. |
| R-04 | **Agent skill stubs deferred boundaries** | aec-orchestrator documents routing logic for ALL 15 skills but marks 6 as "available when dependent package installed". |

**Result**: 15 skills total. 9 in Wave A (buildable now), 6 in Wave B (deferred).

---

## Definitive Skill Inventory (Wave A — 9 skills)

### crosstech-core/ (2 skills)

| Name | Boundary | Research Input | Complexity |
|------|----------|----------------|-----------|
| `crosstech-core-ifc-schema-bridge` | IFC ↔ All formats | vooronderzoek-ifc-ecosystem §1-§5 | L |
| `crosstech-core-coordinate-systems` | BIM local ↔ GIS global | vooronderzoek-bim-gis §1-§7 | L |

### crosstech-impl/ (4 skills)

| Name | Boundary | Research Input | Complexity |
|------|----------|----------------|-----------|
| `crosstech-impl-ifc-erpnext-costing` | IFC ↔ ERPNext | vooronderzoek-aec-automation §1 | M |
| `crosstech-impl-freecad-ifc-bridge` | FreeCAD ↔ IFC | vooronderzoek-aec-automation §2, ifc-ecosystem §4 | M |
| `crosstech-impl-n8n-aec-pipeline` | n8n ↔ AEC tools | vooronderzoek-aec-automation §3 | M |
| `crosstech-impl-docker-aec-stack` | Docker ↔ AEC services | vooronderzoek-aec-automation §4 | M |

### crosstech-errors/ (2 skills)

| Name | Boundary | Research Input | Complexity |
|------|----------|----------------|-----------|
| `crosstech-errors-conversion` | Any ↔ Any | ifc-ecosystem §6, all research | M |
| `crosstech-errors-coordinate-mismatch` | BIM ↔ GIS | bim-gis §6-§7 | M |

### crosstech-agents/ (1 skill)

| Name | Boundary | Research Input | Complexity |
|------|----------|----------------|-----------|
| `crosstech-agents-aec-orchestrator` | Multi-hop | All research, all skills | L |

---

## Batch Execution Plan

| Batch | Skills | Count | Dependencies |
|-------|--------|-------|-------------|
| A1 | core-ifc-schema-bridge, core-coordinate-systems | 2 | None |
| A2 | ifc-erpnext-costing, freecad-ifc-bridge, n8n-aec-pipeline | 3 | Batch A1 |
| A3 | docker-aec-stack, errors-conversion, errors-coordinate-mismatch | 3 | Batch A1 |
| A4 | agents-aec-orchestrator | 1 | All above |

**Total Wave A**: 9 skills across 4 batches.

---

## Constants

```
PROJECT_ROOT = C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package
RESEARCH_IFC = C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\docs\research\vooronderzoek-ifc-ecosystem.md
RESEARCH_BIM_GIS = C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\docs\research\vooronderzoek-bim-gis.md
RESEARCH_AUTOMATION = C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\docs\research\vooronderzoek-aec-automation.md
REQUIREMENTS_FILE = C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\REQUIREMENTS.md
REFERENCE_SKILL = C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package\skills\blender\core\blender-core-api\SKILL.md
```

---

## Per-Skill Agent Prompts

### Batch A1

#### Prompt: crosstech-core-ifc-schema-bridge

```
## Task: Create the crosstech-core-ifc-schema-bridge skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\skills\source\crosstech-core\crosstech-core-ifc-schema-bridge\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (IFC entity mapping tables for all target formats)
3. references/examples.md (working code for reading/writing IFC across tools)
4. references/anti-patterns.md (schema mapping mistakes)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package\skills\blender\core\blender-core-api\SKILL.md

### YAML Frontmatter
---
name: crosstech-core-ifc-schema-bridge
description: >
  Use when mapping IFC entities between different AEC tools: IfcOpenShell (Python),
  web-ifc (WebAssembly), Blender/Bonsai, FreeCAD, Speckle, or Three.js.
  Prevents the common mistake of losing properties, relationships, or geometry
  during IFC data conversion across technology boundaries.
  Covers IFC4/IFC4X3 entity hierarchy, property sets, quantity sets, relationships,
  and how each concept maps to representations in target tools.
  Keywords: IFC, schema, mapping, IfcOpenShell, web-ifc, property sets, quantity sets,
  IfcProduct, IfcRelDefinesByProperties, BIM data conversion, IFC4, IFC4X3.
license: MIT
compatibility: "Designed for Claude Code. Covers IFC4 (ISO 16739-1:2018) and IFC4X3 (ISO 16739-1:2024)."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- IFC schema overview: entity hierarchy from IfcRoot to concrete types
- IFC4 vs IFC4X3 key differences (new infrastructure entities)
- Property sets (IfcPropertySet, IfcPropertySingleValue) — structure and access
- Quantity sets (IfcElementQuantity, IfcQuantityArea/Volume/Length/Count/Weight)
- Relationships (IfcRelAggregates, IfcRelContainedInSpatialStructure, IfcRelDefinesByProperties, IfcRelDefinesByType, IfcRelAssociatesMaterial, IfcRelVoidsElement)
- Type objects (IfcTypeProduct, IfcWallType, etc.)
- Schema mapping table: IFC entity → Blender object, Three.js Object3D, web-ifc flat ID, FreeCAD Part, Speckle Base
- Data loss matrix: what is preserved/lost per target format
- BOTH SIDES: IFC schema concepts AND target format representations

### Research Sections to Read
From vooronderzoek-ifc-ecosystem.md:
- Section 1: IFC Schema Structure (all subsections)
- Section 2: IfcOpenShell Python API (property/quantity access patterns)
- Section 3: web-ifc (property access, differences from IfcOpenShell)
- Section 5: Schema Mapping Between Formats (ALL mapping tables)
- Section 6: Common Integration Failures (property loss, geometry degradation)

### SPECIAL: Technology Boundary Format
This is a CROSS-TECHNOLOGY skill. Structure the main body as:
## Technology Boundary
### Side A: IFC Schema (the universal model)
### Side B: Target Formats (per-tool representation)
### The Bridge: Mapping Rules

### Quality Rules
- English only
- SKILL.md < 500 lines; mapping tables and code go in references/
- Use ALWAYS/NEVER deterministic language
- Specify IFC4 AND IFC4X3 versions explicitly
- Include Critical Warnings section
- Every mapping must note what is LOST in translation
- Delete the .gitkeep file after creating real files
```

#### Prompt: crosstech-core-coordinate-systems

```
## Task: Create the crosstech-core-coordinate-systems skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\skills\source\crosstech-core\crosstech-core-coordinate-systems\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (CRS transformation APIs: pyproj, IfcOpenShell georef, PROJ)
3. references/examples.md (working transformation code examples)
4. references/anti-patterns.md (coordinate system mistakes)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package\skills\blender\core\blender-core-api\SKILL.md

### YAML Frontmatter
---
name: crosstech-core-coordinate-systems
description: >
  Use when transforming coordinates between BIM local systems and GIS global systems,
  handling CRS conversions, or debugging models appearing at wrong locations.
  Prevents the common mistake of applying wrong EPSG codes, ignoring axis conventions
  (Y-up vs Z-up), or losing precision in coordinate transformations.
  Covers IfcMapConversion, IfcProjectedCRS, EPSG codes, pyproj/PROJ transformations,
  and axis conventions for Blender, Three.js, QGIS, Revit, FreeCAD, and IFC.
  Keywords: CRS, EPSG, coordinate system, georeferencing, IfcMapConversion,
  IfcProjectedCRS, pyproj, PROJ, Y-up, Z-up, BIM to GIS, map conversion.
license: MIT
compatibility: "Designed for Claude Code. Covers IFC4/IFC4X3 georeferencing, PROJ 9.x, pyproj 3.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- CRS fundamentals: geographic vs projected, EPSG codes, datum transformations
- Key EPSG codes for AEC: 4326 (WGS84), 28992 (RD New), 7415 (RD+NAP), 32631/32632 (UTM), 3857 (Web Mercator)
- IFC georeferencing: IfcMapConversion attributes, IfcProjectedCRS attributes, transformation formula
- IFC4 vs IFC4X3 georeferencing differences
- Axis conventions table: all major AEC tools (Blender, Three.js, QGIS, IFC, Revit, FreeCAD, Speckle, web-ifc)
- pyproj/PROJ transformation pipeline
- Common failures: Y/Z swap, unit mismatch, missing georef, wrong CRS, true north rotation
- BOTH SIDES: BIM local coordinates AND GIS global coordinates

### Research Sections to Read
From vooronderzoek-bim-gis.md:
- Section 1: CRS Fundamentals (EPSG codes, geographic vs projected)
- Section 2: IFC Georeferencing (IfcMapConversion, IfcProjectedCRS, formulas)
- Section 3: BIM Local Coordinates (per-tool conventions)
- Section 4: PROJ and pyproj (transformation code)
- Section 5: QGIS-BIM Integration patterns
- Section 6: Common Coordinate Failures (all 7 failure modes)
- Section 7: Axis Conventions by Tool (comparison table)

### SPECIAL: Technology Boundary Format
## Technology Boundary
### Side A: BIM Local Coordinates
### Side B: GIS Global Coordinates (CRS-based)
### The Bridge: IfcMapConversion + pyproj Transformation

### Quality Rules
- English only
- SKILL.md < 500 lines; EPSG tables, code, and formulas go in references/
- Use ALWAYS/NEVER deterministic language
- Include the actual IfcMapConversion transformation formula
- Include axis convention comparison table in SKILL.md (compact version)
- Include Critical Warnings with NEVER rules for coordinate mistakes
- Delete the .gitkeep file after creating real files
```

---

### Batch A2

#### Prompt: crosstech-impl-ifc-erpnext-costing

```
## Task: Create the crosstech-impl-ifc-erpnext-costing skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\skills\source\crosstech-impl\crosstech-impl-ifc-erpnext-costing\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (IfcOpenShell quantity API + ERPNext/Frappe REST API)
3. references/examples.md (end-to-end IFC quantity → ERPNext BOM pipeline)
4. references/anti-patterns.md (costing integration mistakes)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package\skills\blender\core\blender-core-api\SKILL.md

### YAML Frontmatter
---
name: crosstech-impl-ifc-erpnext-costing
description: >
  Use when extracting quantities from IFC models and creating cost estimates in ERPNext.
  Prevents the common mistake of mapping IFC quantities to wrong ERPNext fields
  or losing unit information during the IFC-to-BOM conversion.
  Covers IfcElementQuantity extraction via IfcOpenShell, ERPNext BOM structure,
  IFC classification to Item Group mapping, and the complete quantity-to-cost pipeline.
  Keywords: IFC costing, BOM, ERPNext, IfcElementQuantity, quantity takeoff,
  IfcOpenShell, Frappe API, cost estimation, IFC to ERP.
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell 0.8.x and ERPNext v15."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- IFC quantity extraction: IfcElementQuantity, all 5 quantity types, get_psets() API
- ERPNext BOM structure: Item, BOM, BOM Item, Project
- Mapping: IFC classification → ERPNext Item Group, IFC quantities → BOM quantities
- Frappe REST API: creating Items and BOMs programmatically
- Directionality: IFC → ERPNext (primary), ERPNext → IFC (limited, document what's possible)
- Data loss: what IFC info has no ERPNext equivalent
- Unit handling: IFC units (mm, m) → ERPNext UOM

### Research Sections to Read
From vooronderzoek-aec-automation.md:
- Section 1: IFC-to-ERPNext Costing Integration (ALL subsections)
From vooronderzoek-ifc-ecosystem.md:
- Section 1.5: Quantity Sets (IfcElementQuantity structure)
- Section 2: IfcOpenShell API (get_psets, utility modules)

### SPECIAL: Technology Boundary Format
## Technology Boundary
### Side A: IFC Model (IfcOpenShell)
### Side B: ERPNext ERP (Frappe API)
### The Bridge: Quantity Extraction → BOM Creation Pipeline

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Include working Python code for extraction
- Include Frappe API examples for BOM creation
- Document ALL 5 quantity types with examples
- Delete the .gitkeep file
```

#### Prompt: crosstech-impl-freecad-ifc-bridge

```
## Task: Create the crosstech-impl-freecad-ifc-bridge skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\skills\source\crosstech-impl\crosstech-impl-freecad-ifc-bridge\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (FreeCAD BIM/Arch API + IfcOpenShell within FreeCAD)
3. references/examples.md (import/export/round-trip workflows)
4. references/anti-patterns.md (FreeCAD IFC mistakes)

### YAML Frontmatter
---
name: crosstech-impl-freecad-ifc-bridge
description: >
  Use when working with IFC files in FreeCAD, importing BIM models, or doing
  round-trip IFC editing. Prevents the common mistake of using legacy import
  instead of NativeIFC mode or losing IFC data during FreeCAD conversion.
  Covers FreeCAD 1.0+ BIM Workbench, NativeIFC mode, IfcOpenShell integration,
  import/export workflows, and round-trip editing patterns.
  Keywords: FreeCAD, IFC, NativeIFC, BIM Workbench, IfcOpenShell, round-trip,
  FreeCAD BIM, IFC import, IFC export, Arch module.
license: MIT
compatibility: "Designed for Claude Code. Requires FreeCAD 1.0+ with BIM Workbench and IfcOpenShell 0.8.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- FreeCAD BIM Workbench overview (1.0+)
- NativeIFC mode: locked vs unlocked, direct IFC editing
- Traditional import/export vs NativeIFC (decision tree)
- IfcOpenShell as FreeCAD's IFC engine
- Import workflow: IFC → FreeCAD (supported entities, geometry conversion)
- Export workflow: FreeCAD → IFC (preservation, generation)
- Round-trip: open → modify → save back to IFC
- FreeCAD scripting API for IFC: Arch module, BIM module
- Directionality: bidirectional (IFC ↔ FreeCAD)
- NOTE: This skill is MORE SELF-CONTAINED than others because no FreeCAD skill package exists

### Research Sections to Read
From vooronderzoek-aec-automation.md:
- Section 2: FreeCAD-IFC Bridge (ALL subsections)
From vooronderzoek-ifc-ecosystem.md:
- Section 4: FreeCAD IFC Support
- Section 5: Schema Mapping — FreeCAD column

### SPECIAL: Technology Boundary Format
## Technology Boundary
### Side A: FreeCAD (Part objects, BIM Workbench)
### Side B: IFC (IfcOpenShell data model)
### The Bridge: NativeIFC / Traditional Import-Export

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Document NativeIFC as the PREFERRED approach
- Include FreeCAD Python scripting examples
- Delete the .gitkeep file
```

#### Prompt: crosstech-impl-n8n-aec-pipeline

```
## Task: Create the crosstech-impl-n8n-aec-pipeline skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\skills\source\crosstech-impl\crosstech-impl-n8n-aec-pipeline\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (n8n node reference for AEC, API endpoints)
3. references/examples.md (complete workflow JSON snippets)
4. references/anti-patterns.md (n8n AEC pipeline mistakes)

### YAML Frontmatter
---
name: crosstech-impl-n8n-aec-pipeline
description: >
  Use when automating AEC workflows with n8n: IFC processing pipelines,
  Speckle webhook integration, model validation, or connecting AEC tools via APIs.
  Prevents the common mistake of running heavy IFC processing inside n8n nodes
  instead of delegating to external services.
  Covers n8n v1.x nodes for AEC, webhook triggers, Execute Command for IfcOpenShell,
  Speckle/ERPNext API integration, and pipeline patterns.
  Keywords: n8n, workflow automation, AEC pipeline, IFC processing, Speckle webhook,
  ERPNext API, model validation, n8n nodes, Execute Command.
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x with AEC service endpoints."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- n8n v1.x workflow engine overview for AEC use
- Key nodes: HTTP Request, Webhook, Code, Execute Command, FTP/SFTP
- Pipeline pattern 1: IFC upload → validation → property extraction → report
- Pipeline pattern 2: Speckle webhook → change detection → notification
- Pipeline pattern 3: Scheduled BIM check → quality report → email
- Connecting to AEC services: Speckle API, ERPNext API, IfcOpenShell scripts
- Credential management for AEC services
- Directionality: n8n orchestrates AEC tools (n8n → tools, tools → n8n via webhooks)

### Research Sections to Read
From vooronderzoek-aec-automation.md:
- Section 3: n8n Pipeline Automation for AEC (ALL subsections)
- Section 5: Integration Patterns (event-driven, API-based)

### SPECIAL: Technology Boundary Format
## Technology Boundary
### Side A: n8n Workflow Engine
### Side B: AEC Tool APIs (Speckle, ERPNext, IfcOpenShell)
### The Bridge: HTTP/Webhook/CLI Integration

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Use EXACT n8n node names (e.g., n8n-nodes-base.httpRequest)
- Include realistic workflow snippets
- Delete the .gitkeep file
```

---

### Batch A3

#### Prompt: crosstech-impl-docker-aec-stack

```
## Task: Create the crosstech-impl-docker-aec-stack skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\skills\source\crosstech-impl\crosstech-impl-docker-aec-stack\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (Dockerfile patterns, docker-compose services)
3. references/examples.md (complete docker-compose.yml for AEC stacks)
4. references/anti-patterns.md (Docker AEC deployment mistakes)

### YAML Frontmatter
---
name: crosstech-impl-docker-aec-stack
description: >
  Use when containerizing AEC tools or deploying multi-service AEC stacks with Docker.
  Prevents the common mistake of missing geometry dependencies (OCCT/OCC) in
  IfcOpenShell containers or misconfiguring Speckle Server services.
  Covers Dockerfiles for IfcOpenShell, Speckle Server deployment, QGIS Server,
  web-ifc workers, and multi-service docker-compose orchestration.
  Keywords: Docker, AEC, IfcOpenShell container, Speckle Server, QGIS Server,
  docker-compose, web-ifc, containerization, AEC deployment.
license: MIT
compatibility: "Designed for Claude Code. Requires Docker Engine 24+ and Docker Compose v2."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- IfcOpenShell Dockerfile (conda-based with OCCT dependencies)
- Speckle Server docker-compose (6 services: server, frontend, PostgreSQL, Redis, MinIO, webhooks)
- QGIS Server container (qgis/qgis-server:ltr, WMS/WFS)
- web-ifc Node.js worker container
- Multi-service docker-compose for AEC dev environment
- Volume management for large IFC files
- Network architecture: internal vs external
- Resource requirements: memory/CPU for AEC tools
- Directionality: Docker wraps AEC tools (infrastructure layer)

### Research Sections to Read
From vooronderzoek-aec-automation.md:
- Section 4: Docker-Based AEC Service Stack (ALL subsections)
- Section 5: Integration Patterns (service communication)

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Use VERIFIED Docker image names only
- Include resource requirement estimates
- Delete the .gitkeep file
```

#### Prompt: crosstech-errors-conversion

```
## Task: Create the crosstech-errors-conversion skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\skills\source\crosstech-errors\crosstech-errors-conversion\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (diagnostic commands and API calls per tool)
3. references/examples.md (real error messages with solutions)
4. references/anti-patterns.md (conversion mistakes that cause errors)

### YAML Frontmatter
---
name: crosstech-errors-conversion
description: >
  Use when IFC data conversion fails or produces unexpected results: missing geometry,
  lost properties, broken relationships, or schema version mismatches.
  Provides a symptom-to-root-cause decision tree for diagnosing data conversion
  errors across ALL AEC technology boundaries.
  Covers IfcOpenShell, web-ifc, Blender, FreeCAD, Three.js, Speckle, and ERPNext
  conversion failures with specific error messages and fixes.
  Keywords: IFC conversion error, missing geometry, lost properties, schema mismatch,
  BREP to mesh, property loss, encoding error, IFC import failure.
license: MIT
compatibility: "Designed for Claude Code. Covers all AEC technology boundaries."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Symptom-based decision tree: "I see X" → possible causes → fixes
- Property loss: why properties disappear during conversion
- Geometry degradation: BREP → mesh conversion, tessellation quality
- Relationship loss: IfcRelAggregates, spatial containment breaking
- Schema version mismatches: IFC2X3 vs IFC4 vs IFC4X3 incompatibilities
- Encoding issues: UTF-8 in property values, special characters
- Large file handling: memory limits, streaming vs full load
- Partial conversion: some entities convert, others don't

### Research Sections to Read
From vooronderzoek-ifc-ecosystem.md:
- Section 6: Common Integration Failures (ALL 8 scenarios)
- Section 5: Schema Mapping (data loss annotations)
From vooronderzoek-bim-gis.md:
- Section 6: Common Coordinate Failures (coordinate-related conversion errors)

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Decision tree format: symptom → diagnosis → fix
- Include REAL error messages from IfcOpenShell, web-ifc, Blender
- Cover errors from BOTH sides of each boundary
- Delete the .gitkeep file
```

#### Prompt: crosstech-errors-coordinate-mismatch

```
## Task: Create the crosstech-errors-coordinate-mismatch skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\skills\source\crosstech-errors\crosstech-errors-coordinate-mismatch\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (diagnostic APIs for coordinate issues per tool)
3. references/examples.md (real coordinate error scenarios with fixes)
4. references/anti-patterns.md (coordinate handling mistakes)

### YAML Frontmatter
---
name: crosstech-errors-coordinate-mismatch
description: >
  Use when BIM models appear at the wrong location, wrong scale, or wrong orientation
  in GIS or web viewers. Provides a diagnostic decision tree for coordinate-related
  errors: Y-up vs Z-up axis swap, unit mismatches (mm/m/ft), missing georeferencing,
  CRS mismatches, and datum transformation errors.
  Covers coordinate debugging across Blender, Three.js, QGIS, IFC, Revit, and FreeCAD.
  Keywords: wrong location, coordinate mismatch, Y-up Z-up, scale error, EPSG,
  georeferencing missing, axis swap, unit mismatch, CRS error, datum shift.
license: MIT
compatibility: "Designed for Claude Code. Covers all AEC tools with coordinate systems."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Symptom-based decision tree for coordinate errors
- 7 failure modes: Y/Z swap, unit mismatch, missing georef, wrong CRS, true north rotation, UTM zone boundary, vertical datum confusion
- Per-tool axis conventions (compact table)
- Detection methods: how to check coordinates in each tool
- Fix procedures: step-by-step for each failure mode
- Precision issues in coordinate transformations
- Directionality: BIM→GIS and GIS→BIM both covered

### Research Sections to Read
From vooronderzoek-bim-gis.md:
- Section 6: Common Coordinate Failures (ALL 7 failure modes)
- Section 7: Axis Conventions by Tool
- Section 2: IFC Georeferencing (IfcMapConversion for diagnostics)
- Section 4: PROJ and pyproj (for verification code)

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Include axis convention table
- Decision tree format: symptom → diagnosis → fix
- Include verification code (pyproj, IfcOpenShell)
- Delete the .gitkeep file
```

---

### Batch A4

#### Prompt: crosstech-agents-aec-orchestrator

```
## Task: Create the crosstech-agents-aec-orchestrator skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\skills\source\crosstech-agents\crosstech-agents-aec-orchestrator\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (skill routing table, validation methods)
3. references/examples.md (multi-hop pipeline examples)
4. references/anti-patterns.md (orchestration mistakes)

### YAML Frontmatter
---
name: crosstech-agents-aec-orchestrator
description: >
  Use when a user request involves multiple AEC technologies that need to be chained
  together: multi-hop data pipelines (e.g., Revit → Speckle → Blender → IFC → Web),
  technology selection guidance, or cross-tool workflow design.
  Routes to the correct boundary skill(s) for each hop, validates data integrity
  between steps, and handles error recovery.
  Keywords: AEC pipeline, multi-hop, workflow orchestration, technology selection,
  cross-tool integration, BIM pipeline, data integrity validation.
license: MIT
compatibility: "Designed for Claude Code. Orchestrates all Cross-Tech AEC skills."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Skill routing table: user intent → which boundary skill(s) to load
- Decision tree: single-hop vs multi-hop scenarios
- Multi-hop pipeline patterns (A→B→C): how to chain skills
- Data integrity validation between hops
- Error recovery: what to do when a hop fails
- Coverage status: 9 skills available (Wave A), 6 deferred (Wave B — mark as "requires dependent package")
- Pipeline examples: IFC→ERPNext costing, FreeCAD→IFC→web viewer, n8n orchestration

### Research Sections to Read
All three vooronderzoek documents — skim for integration patterns.
From vooronderzoek-aec-automation.md:
- Section 5: Integration Patterns (end-to-end pipeline example)

Read ALL completed Wave A skills in:
C:\Users\Freek Heijting\Documents\GitHub\Cross-Tech-AEC-Claude-Skill-Package\skills\source\

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Include complete routing table
- Mark Wave B skills as "DEFERRED — requires [package name]"
- Include at least 3 multi-hop pipeline examples
- Delete the .gitkeep file
```

---

## Wave B Prompts (DEFERRED)

Wave B skill prompts will be written when dependent packages are complete:
- B1: ifc-to-webifc, ifc-to-threejs, bim-web-viewer (needs ThatOpen + Three.js)
- B2: speckle-blender, speckle-revit, qgis-bim-georef (needs Speckle + QGIS)
