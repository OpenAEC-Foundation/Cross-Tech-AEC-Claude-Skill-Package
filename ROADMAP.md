# Cross-Tech AEC Skill Package — ROADMAP

## Current Status: B7 DONE (Wave A) — Published v0.9.0

> **NOTE:** This package bridges other AEC skill packages. 4 of 8 dependent packages are complete; 4 are skeleton repos. Core skills and automation skills can be built NOW.

| Phase | Description | Status | Skills |
|-------|-------------|--------|--------|
| B0 | Bootstrap (workspace setup) | DONE | — |
| B0.5 | Dependent Package Inventory | DONE | — |
| B1 | Raw Masterplan | DONE | — |
| B2 | Deep Research (Vooronderzoek) | DONE | — |
| B3 | Masterplan Refinement | DONE | — |
| B4 | Topic-Specific Research | SKIPPED (covered in B2) | — |
| B5 | Skill Creation (~5 batches) | DONE (Wave A) | 9/9 Wave A |
| B6 | Validation | DONE (Wave A) | 9/9 PASS |
| B7 | Publication | DONE (Wave A) | v0.9.0 released |

## Skill Inventory (preliminary): ~15 skills

| Category | Skills | Count |
|----------|--------|-------|
| core/ | ifc-schema-bridge, coordinate-systems | 2 |
| impl/ | ifc-to-webifc, ifc-to-threejs, speckle-blender, speckle-revit, qgis-bim-georef, ifc-erpnext-costing, bim-web-viewer, freecad-ifc-bridge, n8n-aec-pipeline, docker-aec-stack | 10 |
| errors/ | conversion, coordinate-mismatch | 2 |
| agents/ | aec-orchestrator | 1 |

## Dependent Package Inventory (B0.5 — verified 2026-03-20)

| Package | Status | Skills | Buildable Now? |
|---------|--------|--------|---------------|
| [x] Blender-Bonsai-IfcOpenShell-Sverchok | **COMPLETE** | 73 | Yes |
| [ ] Speckle-Claude-Skill-Package | SKELETON | 0 | No — blocks speckle-blender, speckle-revit |
| [ ] QGIS-Claude-Skill-Package | SKELETON | 0 | No — blocks qgis-bim-georef |
| [ ] Three.js-Claude-Skill-Package | SKELETON | 0 | No — blocks ifc-to-threejs, bim-web-viewer |
| [ ] ThatOpen-Claude-Skill-Package | SKELETON | 0 | No — blocks ifc-to-webifc, bim-web-viewer |
| [x] ERPNext_Anthropic_Claude_Development | **COMPLETE** | 28 | Yes |
| [x] n8n-Claude-Skill-Package | **COMPLETE** | 21 | Yes |
| [x] Docker-Claude-Skill-Package | **COMPLETE** | — | Yes |
| [ ] FreeCAD-Claude-Skill-Package | DOES NOT EXIST | 0 | Partial — IfcOpenShell side available |

### Buildability Assessment

**CAN BUILD NOW (9 skills):**
- core/ifc-schema-bridge — independent
- core/coordinate-systems — independent
- impl/ifc-erpnext-costing — IfcOpenShell + ERPNext both complete
- impl/freecad-ifc-bridge — IfcOpenShell side available, FreeCAD docs sufficient
- impl/n8n-aec-pipeline — n8n complete
- impl/docker-aec-stack — Docker complete
- errors/conversion — depends on core only
- errors/coordinate-mismatch — depends on core only
- agents/aec-orchestrator — orchestration logic, can stub deferred boundaries

**DEFERRED (6 skills — waiting on dependent packages):**
- impl/ifc-to-webifc — needs ThatOpen package
- impl/ifc-to-threejs — needs Three.js package
- impl/speckle-blender — needs Speckle package
- impl/speckle-revit — needs Speckle package
- impl/qgis-bim-georef — needs QGIS package
- impl/bim-web-viewer — needs ThatOpen + Three.js packages
