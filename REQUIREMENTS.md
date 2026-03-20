# Cross-Tech AEC Skill Package — REQUIREMENTS

## Quality Guarantees

Every skill in this package MUST meet the following criteria:

### 1. Format-correct
- SKILL.md < 500 lines
- Valid YAML frontmatter with `name` and `description`
- `name`: kebab-case, max 64 characters, prefix `crosstech-`
- `description`: max 1024 characters, includes trigger keywords
- `references/` directory with: methods.md, examples.md, anti-patterns.md

### 2. Boundary-documented (SPECIAL REQUIREMENT)
- Every skill MUST document BOTH sides of the technology boundary
- Side A: technology name, version, data format, API surface
- Side B: technology name, version, data format, API surface
- The Bridge: how data/models/coordinates transform between A and B
- Failure modes: what goes wrong at the boundary and why

### 3. Version-explicit
- Specify versions of BOTH technologies in the boundary
- IFC: IFC4 (ISO 16739-1:2018) or IFC4X3 (ISO 16739-1:2024)
- CRS: EPSG codes for coordinate reference systems
- Note breaking changes between versions where applicable

### 4. Anti-pattern-free
- Every skill MUST document known integration mistakes in references/anti-patterns.md
- Common data loss scenarios at boundaries documented
- Coordinate transformation pitfalls documented
- Schema mismatch patterns documented

### 5. Deterministic
- Use ALWAYS/NEVER language
- No hedging words: "might", "consider", "often", "usually"
- Decision trees for conditional logic

### 6. Self-contained
- Each skill works independently without requiring other skills
- All necessary context included within the skill
- Cross-references to related skills and dependent packages are informational only

### 7. English-only
- All skill content in English
- Claude reads English, responds in any language
- No Dutch, German, or other languages in skill files

## Per-Category Requirements

### Core Skills
- MUST cover the fundamental concepts that ALL boundary skills depend on
- crosstech-core-ifc-schema-bridge: MUST document IFC schema mapping to ALL target formats
- crosstech-core-coordinate-systems: MUST cover CRS transformations, EPSG codes, and BIM local coordinates

### Implementation Skills
- MUST provide working code/config for crossing the specific boundary
- MUST include realistic examples (not just toy examples)
- MUST document data loss: what information is preserved and what is lost in translation
- MUST specify directionality: A→B, B→A, or bidirectional

### Error Skills
- MUST include real error scenarios with reproduction steps
- MUST provide diagnostic decision trees
- MUST cover errors from BOTH sides of the boundary

### Agent Skills
- MUST include decision trees for selecting the right boundary skill
- MUST handle multi-hop scenarios (A→B→C)
- MUST validate data integrity after boundary crossing

## Standards Compliance

| Standard | Scope | Skills |
|----------|-------|--------|
| IFC4 (ISO 16739-1:2018) | Building data model | All IFC-related skills |
| IFC4X3 (ISO 16739-1:2024) | Infrastructure extension | All IFC-related skills |
| EPSG Registry | Coordinate reference systems | coordinate-systems, qgis-bim-georef |
| buildingSMART Data Dictionary | Property definitions | ifc-schema-bridge |
| OGC Standards | Geospatial interoperability | qgis-bim-georef |
