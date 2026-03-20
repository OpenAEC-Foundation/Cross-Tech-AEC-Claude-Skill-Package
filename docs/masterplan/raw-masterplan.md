# Cross-Tech AEC Skill Package — Raw Masterplan (B1)

## Status
Phase B1 — Raw masterplan. Created 2026-03-20.

---

## Technology Boundary Map

This package covers the integration points where AEC technologies meet. Each cell represents a potential skill.

```
                 Blender  IfcOpenShell  Speckle  QGIS  Three.js  web-ifc  FreeCAD  ERPNext  n8n  Docker
Blender            —         (pkg)       S5       —      —         —        —        —       —     —
IfcOpenShell      (pkg)       —          —        —      —         S3       S10      S8      —     S12
Speckle            S5         —          —        —      —         —        —        —       S11   S12
QGIS               —         S7         —        —      —         —        —        —       —     S12
Three.js           —         —          —        —      —         S4       —        —       —     —
web-ifc            —         S3         —        —      S4        —        —        —       —     S12
FreeCAD            —         S10        —        —      —         —        —        —       —     —
ERPNext            —         S8         —        —      —         —        —        —       S11   S12
n8n                —         —          S11      —      —         —        —        S11     —     S12
Docker             —         S12        S12      S12    —         S12      —        S12     S12   —

(pkg) = covered by single-technology package
SN = skill number in this package
```

### Boundary Categories

| Category | Boundaries | Why AI Fails Here |
|----------|-----------|-------------------|
| **Data Format** | IFC↔web-ifc, IFC↔Three.js mesh, IFC↔FreeCAD | Different internal representations of the same BIM data. Property loss, geometry degradation, schema version mismatches. |
| **Coordinate** | BIM local↔GIS global, Y-up↔Z-up, mm↔m↔ft | Silent coordinate corruption. Models appear at wrong location, wrong scale, wrong orientation. No error thrown. |
| **Transport** | Speckle↔Blender, Speckle↔Revit | Object conversion between Speckle's Base objects and native application objects. Material loss, relationship loss. |
| **Semantic** | IFC quantities↔ERPNext BOMs | Mapping domain concepts: IFC IfcQuantityArea to ERPNext BOM Item quantity. Different classification systems. |
| **Orchestration** | n8n↔AEC tools, Docker↔AEC services | Automating cross-tool workflows. API version drift, container networking, webhook reliability. |

---

## Preliminary Skill Inventory (15 skills)

### Tier 1: Foundation (can build NOW — no dependencies)

| # | Skill | Category | Boundary | Complexity |
|---|-------|----------|----------|-----------|
| 1 | crosstech-core-ifc-schema-bridge | core | IFC ↔ All formats | L |
| 2 | crosstech-core-coordinate-systems | core | BIM local ↔ GIS global | L |

### Tier 2: Buildable NOW (dependent packages complete)

| # | Skill | Category | Boundary | Dep. Packages |
|---|-------|----------|----------|--------------|
| 8 | crosstech-impl-ifc-erpnext-costing | impl | IFC ↔ ERPNext | Blender-Bonsai ✓, ERPNext ✓ |
| 10 | crosstech-impl-freecad-ifc-bridge | impl | FreeCAD ↔ IFC | Blender-Bonsai ✓ (no FreeCAD pkg) |
| 11 | crosstech-impl-n8n-aec-pipeline | impl | n8n ↔ AEC tools | n8n ✓ |
| 12 | crosstech-impl-docker-aec-stack | impl | Docker ↔ AEC services | Docker ✓ |

### Tier 3: Buildable after core (depends on core skills only)

| # | Skill | Category | Boundary | Dependencies |
|---|-------|----------|----------|-------------|
| 13 | crosstech-errors-conversion | errors | Any ↔ Any | core-ifc-schema-bridge |
| 14 | crosstech-errors-coordinate-mismatch | errors | BIM ↔ GIS | core-coordinate-systems |
| 15 | crosstech-agents-aec-orchestrator | agents | Multi-hop | All impl skills |

### Tier 4: DEFERRED (dependent packages incomplete)

| # | Skill | Category | Boundary | Blocked By |
|---|-------|----------|----------|-----------|
| 3 | crosstech-impl-ifc-to-webifc | impl | IfcOpenShell ↔ web-ifc | ThatOpen (skeleton) |
| 4 | crosstech-impl-ifc-to-threejs | impl | IFC ↔ Three.js | Three.js (skeleton) |
| 5 | crosstech-impl-speckle-blender | impl | Speckle ↔ Blender | Speckle (skeleton) |
| 6 | crosstech-impl-speckle-revit | impl | Speckle ↔ Revit | Speckle (skeleton) |
| 7 | crosstech-impl-qgis-bim-georef | impl | QGIS ↔ BIM/IFC | QGIS (skeleton) |
| 9 | crosstech-impl-bim-web-viewer | impl | BIM/IFC ↔ Web browser | ThatOpen + Three.js (skeleton) |

---

## Build Order

### Wave A: Core + Automation (9 skills — buildable NOW)

| Batch | Skills | Count | Strategy |
|-------|--------|-------|----------|
| A1 | core-ifc-schema-bridge, core-coordinate-systems | 2 | Foundation — no deps, parallel |
| A2 | ifc-erpnext-costing, freecad-ifc-bridge, n8n-aec-pipeline | 3 | Impl skills with complete deps, parallel |
| A3 | docker-aec-stack, errors-conversion, errors-coordinate-mismatch | 3 | Remaining buildable skills, parallel |
| A4 | agents-aec-orchestrator | 1 | Orchestrator — references all skills |

### Wave B: Deferred Skills (6 skills — when packages are ready)

| Batch | Skills | Count | Unblocked When |
|-------|--------|-------|---------------|
| B1 | ifc-to-webifc, ifc-to-threejs, bim-web-viewer | 3 | ThatOpen + Three.js packages complete |
| B2 | speckle-blender, speckle-revit, qgis-bim-georef | 3 | Speckle + QGIS packages complete |

---

## Research Plan

### Deep Research (Vooronderzoek) — needed for Wave A

| Document | Covers Skills | Min Words |
|----------|--------------|-----------|
| vooronderzoek-ifc-ecosystem.md | #1 (ifc-schema-bridge), #10 (freecad-ifc-bridge), #13 (errors-conversion) | 2000 |
| vooronderzoek-bim-gis.md | #2 (coordinate-systems), #14 (errors-coordinate-mismatch) | 2000 |
| vooronderzoek-aec-automation.md | #8 (ifc-erpnext-costing), #11 (n8n-aec-pipeline), #12 (docker-aec-stack) | 2000 |

### Deferred Research (for Wave B)

| Document | Covers Skills | Blocked By |
|----------|--------------|-----------|
| vooronderzoek-web-visualization.md | #3, #4, #9 | ThatOpen + Three.js packages |
| vooronderzoek-speckle-transport.md | #5, #6 | Speckle package |
| vooronderzoek-qgis-bim.md | #7 | QGIS package |

---

## Key Risks

1. **Skeleton packages may change scope** — skills bridging to them may need revision
2. **FreeCAD has no dedicated package** — freecad-ifc-bridge must be more self-contained
3. **IFC4X3 is still gaining adoption** — skills must cover both IFC4 and IFC4X3
4. **Coordinate systems are the #1 silent failure** — extra research investment needed

---

## Next Steps

1. ~~B0.5: Package inventory~~ DONE
2. **B1: Raw masterplan** ← THIS DOCUMENT
3. **B2: Deep research** — 3 vooronderzoek documents for Wave A skills
4. **B3: Masterplan refinement** — finalize skill specs and agent prompts
5. **B4-B5: Research + skill creation** — Wave A batches A1–A4
