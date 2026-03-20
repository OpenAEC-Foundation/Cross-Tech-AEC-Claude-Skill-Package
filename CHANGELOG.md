# Changelog

All notable changes to the Cross-Tech AEC Skill Package will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

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
