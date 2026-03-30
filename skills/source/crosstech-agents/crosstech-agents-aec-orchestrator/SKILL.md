---
name: crosstech-agents-aec-orchestrator
description: >
  Use when a user request involves multiple AEC technologies that need to be chained
  together: multi-hop data pipelines (e.g., Revit → Speckle → Blender → IFC → Web),
  technology selection guidance, or cross-tool workflow design.
  Routes to the correct boundary skill(s) for each hop, validates data integrity
  between steps, and handles error recovery.
  Keywords: AEC pipeline, multi-hop, workflow orchestration, technology selection,
  cross-tool integration, BIM pipeline, data integrity validation,
  connect AEC tools, chain BIM workflow, which tool to use.
license: MIT
compatibility: "Designed for Claude Code. Orchestrates all Cross-Tech AEC skills."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-agents-aec-orchestrator

## Quick Reference

### Routing Summary

| User Intent | Skill(s) to Load | Hop Count |
|-------------|-------------------|-----------|
| Map IFC entities between tools | `crosstech-core-ifc-schema-bridge` | 1 |
| Fix coordinate / georef issues | `crosstech-core-coordinate-systems` | 1 |
| Extract BIM costs to ERPNext | `crosstech-impl-ifc-erpnext-costing` | 1 |
| Edit IFC in FreeCAD | `crosstech-impl-freecad-ifc-bridge` | 1 |
| Automate AEC workflows with n8n | `crosstech-impl-n8n-aec-pipeline` | 1 |
| Containerize AEC tools | `crosstech-impl-docker-aec-stack` | 1 |
| Diagnose conversion errors | `crosstech-errors-conversion` | 1 |
| Diagnose coordinate errors | `crosstech-errors-coordinate-mismatch` | 1 |
| FreeCAD → IFC → ERPNext costing | `freecad-ifc-bridge` → `ifc-erpnext-costing` | 2 |
| IFC → Docker processing → n8n | `docker-aec-stack` → `n8n-aec-pipeline` | 2 |
| Full BIM pipeline (multi-hop) | See Multi-Hop Pipeline Patterns | 3+ |

### Critical Warnings

**NEVER** skip the data integrity validation between hops — silent data loss at boundaries is the #1 cause of pipeline failures.

**NEVER** chain more than 4 hops without intermediate validation checkpoints — error propagation makes debugging impossible.

**NEVER** assume a single skill covers a multi-technology request — ALWAYS decompose into individual boundary crossings first.

**ALWAYS** load `crosstech-core-ifc-schema-bridge` when ANY hop involves IFC data.

**ALWAYS** load `crosstech-core-coordinate-systems` when ANY hop involves georeferenced data or coordinate transformations.

**ALWAYS** load the relevant `crosstech-errors-*` skill when a hop fails — do NOT guess at the fix.

---

## Skill Routing Table

### Wave A: Available Skills (9 skills)

| # | Skill | Category | Boundary | Load When |
|---|-------|----------|----------|-----------|
| 1 | `crosstech-core-ifc-schema-bridge` | core | IFC ↔ All formats | User maps IFC entities between ANY tools, checks schema versions, or queries properties across tool boundaries |
| 2 | `crosstech-core-coordinate-systems` | core | BIM local ↔ GIS global | User transforms coordinates, adds georeferencing, fixes location issues, or moves data between Z-up and Y-up systems |
| 3 | `crosstech-impl-ifc-erpnext-costing` | impl | IFC ↔ ERPNext | User extracts quantities from IFC for cost estimation, maps property sets to ERPNext doctypes, or builds BOM from BIM |
| 4 | `crosstech-impl-freecad-ifc-bridge` | impl | FreeCAD ↔ IFC | User imports/exports IFC in FreeCAD, uses NativeIFC mode, or round-trip edits IFC files in FreeCAD |
| 5 | `crosstech-impl-n8n-aec-pipeline` | impl | n8n ↔ AEC tools | User automates AEC workflows, sets up IFC processing triggers, Speckle webhooks, or model validation pipelines |
| 6 | `crosstech-impl-docker-aec-stack` | impl | Docker ↔ AEC services | User containerizes IfcOpenShell, deploys Speckle server, runs QGIS server, or builds multi-service AEC stacks |
| 7 | `crosstech-errors-conversion` | errors | Any ↔ Any format | User encounters missing geometry, lost properties, broken relationships, schema mismatches, or partial conversion failures |
| 8 | `crosstech-errors-coordinate-mismatch` | errors | BIM ↔ GIS | User sees models at wrong location, scale mismatches, flipped axes, missing georeferencing, or CRS errors |
| 9 | `crosstech-agents-aec-orchestrator` | agents | Multi-hop | (This skill) User needs multi-technology pipeline design or technology selection guidance |

### Wave B: Deferred Skills (6 skills)

| # | Skill | Boundary | Status | Requires Package |
|---|-------|----------|--------|------------------|
| 10 | `crosstech-impl-ifc-to-webifc` | IfcOpenShell ↔ web-ifc | DEFERRED | ThatOpen-Claude-Skill-Package |
| 11 | `crosstech-impl-ifc-to-threejs` | IFC ↔ Three.js | DEFERRED | Three.js-Claude-Skill-Package |
| 12 | `crosstech-impl-speckle-blender` | Speckle ↔ Blender | DEFERRED | Speckle-Claude-Skill-Package |
| 13 | `crosstech-impl-speckle-revit` | Speckle ↔ Revit | DEFERRED | Speckle-Claude-Skill-Package |
| 14 | `crosstech-impl-qgis-bim-georef` | QGIS ↔ BIM/IFC | DEFERRED | QGIS-Claude-Skill-Package |
| 15 | `crosstech-impl-bim-web-viewer` | BIM/IFC ↔ Web browser | DEFERRED | ThatOpen-Claude-Skill-Package + Three.js-Claude-Skill-Package |

**ALWAYS** inform the user when a required skill is deferred and state which package must be installed.

---

## Decision Tree: Single-Hop vs Multi-Hop

```
START: User request involves AEC technologies
│
├── Q1: How many technology boundaries are crossed?
│   ├── ONE boundary → Single-hop. Load the matching skill from the routing table.
│   └── TWO or more boundaries → Multi-hop. Continue to Q2.
│
├── Q2: Decompose into individual boundary crossings.
│   ├── List each hop: A → B → C → ...
│   ├── For EACH hop, identify the matching boundary skill.
│   └── Are ALL required skills available (Wave A)?
│       ├── YES → Proceed to Multi-Hop Pipeline Patterns.
│       └── NO → Identify which skills are DEFERRED (Wave B).
│           ├── Inform the user which packages must be installed.
│           └── Provide the available hops and mark gaps.
│
├── Q3: Does ANY hop involve IFC data?
│   ├── YES → ALWAYS also load crosstech-core-ifc-schema-bridge.
│   └── NO → Skip.
│
├── Q4: Does ANY hop involve coordinate transformations?
│   ├── YES → ALWAYS also load crosstech-core-coordinate-systems.
│   └── NO → Skip.
│
└── Q5: Is this a new pipeline design or debugging an existing one?
    ├── NEW DESIGN → Follow Multi-Hop Pipeline Patterns below.
    └── DEBUGGING → Load the relevant crosstech-errors-* skill first.
```

---

## Multi-Hop Pipeline Patterns

### Pattern 1: IFC → ERPNext Costing Pipeline

**Scenario**: Extract quantities from an IFC model and generate cost estimates in ERPNext.

```
IFC File
  │
  ├── Hop 1: Parse IFC structure
  │   Skill: crosstech-core-ifc-schema-bridge
  │   Action: Identify IfcQuantityArea, IfcQuantityVolume, IfcQuantityLength
  │   Validate: Schema version matches tool support, quantities exist
  │
  └── Hop 2: Map quantities to ERPNext
      Skill: crosstech-impl-ifc-erpnext-costing
      Action: Map property sets to BOM items, create cost estimate
      Validate: All mapped fields have values, unit conversions applied
```

**Integrity checks between hops**:
1. Verify IFC schema version is supported (IFC4 or IFC4X3)
2. Confirm quantity sets exist on target elements (`IfcElementQuantity` attached)
3. Verify unit consistency — IFC units MUST be converted before ERPNext mapping

### Pattern 2: FreeCAD → IFC → Docker Processing

**Scenario**: Author a BIM model in FreeCAD, export to IFC, process in a Docker container.

```
FreeCAD Model
  │
  ├── Hop 1: Export IFC from FreeCAD
  │   Skill: crosstech-impl-freecad-ifc-bridge
  │   Action: Use NativeIFC mode, export to IFC4
  │   Validate: Exported IFC opens in IfcOpenShell without errors
  │
  ├── Hop 2: Containerize processing
  │   Skill: crosstech-impl-docker-aec-stack
  │   Action: Run IfcOpenShell validation in Docker container
  │   Validate: Container exits with code 0, output IFC is valid
  │
  └── Hop 3 (optional): Automate with n8n
      Skill: crosstech-impl-n8n-aec-pipeline
      Action: Trigger Docker processing on file upload
      Validate: n8n workflow completes, output file accessible
```

**Integrity checks between hops**:
1. After FreeCAD export: verify `model.schema` returns expected version
2. After Docker processing: verify file size is non-zero, schema unchanged
3. After n8n trigger: verify HTTP status 200, output path exists

### Pattern 3: Full AEC Pipeline (n8n Orchestrated)

**Scenario**: Automated pipeline — IFC uploaded, validated, quantities extracted, costs pushed to ERPNext.

```
IFC Upload (webhook)
  │
  ├── Hop 1: n8n receives webhook trigger
  │   Skill: crosstech-impl-n8n-aec-pipeline
  │   Action: Webhook node receives file, stores to volume
  │   Validate: File exists on disk, size > 0
  │
  ├── Hop 2: Docker validates IFC
  │   Skill: crosstech-impl-docker-aec-stack
  │   Action: IfcOpenShell container validates schema + geometry
  │   Validate: Validation report has zero critical errors
  │
  ├── Hop 3: Parse IFC structure
  │   Skill: crosstech-core-ifc-schema-bridge
  │   Action: Extract entity hierarchy, property sets, quantities
  │   Validate: Expected entity types found, quantities non-empty
  │
  └── Hop 4: Push costs to ERPNext
      Skill: crosstech-impl-ifc-erpnext-costing
      Action: Map quantities → BOM items → cost estimate
      Validate: ERPNext API returns 200, estimate ID received
```

**Integrity checks between hops**:
1. After upload: file hash matches sender hash (no corruption)
2. After validation: zero CRITICAL findings, WARNINGS logged
3. After parsing: element count > 0, at least one quantity set found
4. After cost push: ERPNext response contains valid document name

---

## Data Integrity Validation

### Between-Hop Validation Checklist

ALWAYS run these checks after EACH boundary crossing:

| Check | What to Verify | Failure Action |
|-------|---------------|----------------|
| **Schema version** | Output schema matches expected version | Load `crosstech-errors-conversion` |
| **Entity count** | Element count in output >= count in input (no silent drops) | Load `crosstech-errors-conversion` |
| **Property preservation** | Sample 3 random elements — verify properties survived | Load `crosstech-errors-conversion` |
| **Geometry presence** | All elements with geometry in input have geometry in output | Load `crosstech-errors-conversion` |
| **Unit consistency** | Units in output match the expected target units | Load `crosstech-core-ifc-schema-bridge` |
| **Coordinate validity** | Coordinates fall within valid CRS range | Load `crosstech-errors-coordinate-mismatch` |
| **Relationship integrity** | Spatial hierarchy preserved (site → building → storey) | Load `crosstech-core-ifc-schema-bridge` |
| **File validity** | Output file is non-zero size and parseable | Load `crosstech-errors-conversion` |

### Validation Code Pattern

```python
def validate_hop(input_model, output_model, hop_name: str) -> list[str]:
    """ALWAYS call between pipeline hops. Returns list of errors (empty = pass)."""
    errors = []

    # Schema version
    if input_model.schema != output_model.schema:
        errors.append(f"[{hop_name}] Schema changed: {input_model.schema} → {output_model.schema}")

    # Entity count
    input_count = len(input_model.by_type("IfcProduct"))
    output_count = len(output_model.by_type("IfcProduct"))
    if output_count < input_count:
        errors.append(f"[{hop_name}] Entity loss: {input_count} → {output_count} ({input_count - output_count} lost)")

    # Property spot check (3 random elements)
    import random
    products = output_model.by_type("IfcProduct")
    if products:
        samples = random.sample(products, min(3, len(products)))
        for elem in samples:
            psets = ifcopenshell.util.element.get_psets(elem)
            if not psets:
                errors.append(f"[{hop_name}] {elem.is_a()} #{elem.id()} has no properties")

    return errors
```

---

## Error Recovery

When a hop fails, follow this procedure:

```
HOP FAILED
│
├── Step 1: Identify the failure type
│   ├── Conversion error (missing geometry, lost properties, schema mismatch)
│   │   └── Load: crosstech-errors-conversion
│   ├── Coordinate error (wrong location, scale, axis, CRS)
│   │   └── Load: crosstech-errors-coordinate-mismatch
│   └── Tool-specific error (API error, timeout, auth failure)
│       └── Load the impl/ skill for that specific boundary
│
├── Step 2: Diagnose using the loaded error skill
│   └── Follow the symptom → root cause decision tree in that skill
│
├── Step 3: Fix and retry the failed hop ONLY
│   └── NEVER restart the entire pipeline — resume from the failed hop
│
└── Step 4: Re-run validation on the fixed hop output
    └── ONLY proceed to the next hop after validation passes
```

**NEVER** retry a failed hop more than 3 times without human review.

**ALWAYS** log the error details before retrying — silent retries mask root causes.

---

## Critical Rules

1. **ALWAYS** decompose multi-technology requests into individual boundary crossings before selecting skills.
2. **ALWAYS** load `crosstech-core-ifc-schema-bridge` when ANY hop in the pipeline involves IFC data.
3. **ALWAYS** load `crosstech-core-coordinate-systems` when ANY hop involves georeferenced data.
4. **ALWAYS** validate data integrity between every hop using the validation checklist.
5. **ALWAYS** inform the user when a required skill is deferred (Wave B) and name the missing package.
6. **ALWAYS** load the relevant `crosstech-errors-*` skill when a hop fails — do NOT guess at fixes.
7. **NEVER** chain more than 4 hops without intermediate validation checkpoints.
8. **NEVER** skip between-hop validation — silent data loss at boundaries is the #1 pipeline failure cause.
9. **NEVER** retry a failed hop more than 3 times without escalating to human review.
10. **NEVER** assume two technologies use the same units, axis conventions, or schema versions.

---

## Deferred Skills (Wave B)

The following 6 skills are planned but NOT YET AVAILABLE. They require their respective skill packages to be installed first.

| Skill | Boundary | Required Package | Workaround |
|-------|----------|-----------------|------------|
| `crosstech-impl-ifc-to-webifc` | IfcOpenShell ↔ web-ifc | ThatOpen-Claude-Skill-Package | Use `crosstech-core-ifc-schema-bridge` for schema mapping, implement web-ifc calls manually |
| `crosstech-impl-ifc-to-threejs` | IFC ↔ Three.js | Three.js-Claude-Skill-Package | Use `crosstech-core-ifc-schema-bridge` + `coordinate-systems` for axis swap, implement Three.js loader manually |
| `crosstech-impl-speckle-blender` | Speckle ↔ Blender | Speckle-Claude-Skill-Package | Use Speckle connector documentation directly |
| `crosstech-impl-speckle-revit` | Speckle ↔ Revit | Speckle-Claude-Skill-Package | Use Speckle connector documentation directly |
| `crosstech-impl-qgis-bim-georef` | QGIS ↔ BIM/IFC | QGIS-Claude-Skill-Package | Use `crosstech-core-coordinate-systems` for CRS math, import into QGIS manually |
| `crosstech-impl-bim-web-viewer` | BIM/IFC ↔ Web | ThatOpen + Three.js packages | Combine `ifc-schema-bridge` + `coordinate-systems`, build viewer manually |

---

## Reference Links

- [references/methods.md](references/methods.md) — Skill routing table details, validation method signatures
- [references/examples.md](references/examples.md) — Complete multi-hop pipeline examples with code
- [references/anti-patterns.md](references/anti-patterns.md) — Orchestration mistakes and how to avoid them

### Related Skills (Cross-References)

- `crosstech-core-ifc-schema-bridge` — Foundation for ALL IFC-related hops
- `crosstech-core-coordinate-systems` — Foundation for ALL georeferenced hops
- `crosstech-errors-conversion` — First response for conversion failures
- `crosstech-errors-coordinate-mismatch` — First response for coordinate failures
