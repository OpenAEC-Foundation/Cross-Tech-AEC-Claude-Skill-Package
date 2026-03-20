# Changelog

All notable changes to the Cross-Tech AEC Skill Package will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.0.0] - 2026-03-20

### Added
- **6 Wave B skills** completing the full package (15/15):
  - `crosstech-impl-ifc-to-webifc` (349 lines) — IfcOpenShell vs web-ifc API bridging
  - `crosstech-impl-ifc-to-threejs` (463 lines) — IFC to Three.js scene rendering
  - `crosstech-impl-bim-web-viewer` (363 lines) — End-to-end BIM web viewer pipeline
  - `crosstech-impl-speckle-blender` (374 lines) — Speckle↔Blender data transport
  - `crosstech-impl-speckle-revit` (293 lines) — Speckle↔Revit connector patterns
  - `crosstech-impl-qgis-bim-georef` (435 lines) — QGIS↔BIM georeferencing
- **2 additional research documents**:
  - `vooronderzoek-web-visualization.md` (~3,500 words)
  - `vooronderzoek-speckle-qgis.md` (4,543 words)
- Social preview banner (`docs/social-preview-banner.html`)
- Wave B validation report appended

## [0.9.0] - 2026-03-20

### Added
- **9 Wave A skills** — all validated and passing quality checks:
  - `crosstech-core-ifc-schema-bridge` (266 lines) — IFC schema mapping to all target formats
  - `crosstech-core-coordinate-systems` (289 lines) — CRS, georeferencing, axis conventions
  - `crosstech-impl-ifc-erpnext-costing` (363 lines) — IFC quantity extraction to ERPNext BOMs
  - `crosstech-impl-freecad-ifc-bridge` (357 lines) — FreeCAD NativeIFC workflows
  - `crosstech-impl-n8n-aec-pipeline` (309 lines) — n8n workflow automation for AEC
  - `crosstech-impl-docker-aec-stack` (330 lines) — Docker containerization for AEC tools
  - `crosstech-errors-conversion` (381 lines) — Cross-technology conversion error diagnosis
  - `crosstech-errors-coordinate-mismatch` (425 lines) — Coordinate mismatch debugging
  - `crosstech-agents-aec-orchestrator` (328 lines) — Multi-hop pipeline orchestration
- **3 deep research documents** (~12,800 words total):
  - `vooronderzoek-ifc-ecosystem.md` (4,349 words)
  - `vooronderzoek-bim-gis.md` (4,390 words)
  - `vooronderzoek-aec-automation.md` (4,054 words)
- **Definitive masterplan** with agent prompts for all 9 skills
- **Validation report** — all 9 skills PASS
- **README.md** with installation instructions and skill table
- Dependent package inventory (B0.5): 4 complete, 4 skeleton, 1 missing
- Decisions D-006 (two-wave strategy), D-007 (FreeCAD self-contained)
- Lesson L-016 (dependent package inventory critical for cross-tech)

### Deferred
- 6 Wave B skills waiting on dependent packages (Speckle, QGIS, Three.js, ThatOpen)

## [0.0.1] - 2026-03-20

### Added
- Bootstrap workspace with all governance files
- CLAUDE.md with 8 protocols and technology scope
- ROADMAP.md with 7-phase plan
- WAY_OF_WORK.md with 7-phase research-first methodology
- REQUIREMENTS.md with quality guarantees including boundary documentation requirement
- DECISIONS.md with D-001 to D-005
- LESSONS.md with L-001 to L-015 inherited from other packages
- SOURCES.md with references to all dependent AEC skill packages and standards
- INDEX.md with preliminary catalog of ~15 skills
