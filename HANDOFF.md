# Handoff: Cross-Tech-AEC-Claude-Skill-Package

> Laatste update: 2026-03-20

## Status
- **Fase:** B6 DONE — Wave A gevalideerd, klaar voor publicatie
- **Skills:** 9/15 geschreven en gevalideerd (Wave A)
- **GitHub remote:** https://github.com/OpenAEC-Foundation/Cross-Tech-AEC-Claude-Skill-Package
- **README:** Ontbreekt nog (onderdeel van B7)

## Wat er is

### Governance docs (compleet)
CLAUDE.md, ROADMAP.md, INDEX.md, WAY_OF_WORK.md, DECISIONS.md (D-001 t/m D-007), LESSONS.md (L-001 t/m L-016), SOURCES.md, REQUIREMENTS.md, CHANGELOG.md, START-PROMPT.md, OPEN-QUESTIONS.md

### Research (compleet voor Wave A)
- `docs/research/vooronderzoek-ifc-ecosystem.md` — 4.349 woorden
- `docs/research/vooronderzoek-bim-gis.md` — 4.390 woorden
- `docs/research/vooronderzoek-aec-automation.md` — 4.054 woorden

### Masterplan
- `docs/masterplan/raw-masterplan.md` — twee-wave strategie
- `docs/masterplan/crosstech-masterplan.md` — definitief met agent prompts

### Skills (9/15 — Wave A compleet)
Alle 9 skills gevalideerd (0 failures):

| # | Skill | Lines | Status |
|---|-------|-------|--------|
| 1 | crosstech-core-ifc-schema-bridge | 266 | PASS |
| 2 | crosstech-core-coordinate-systems | 289 | PASS |
| 3 | crosstech-impl-ifc-erpnext-costing | 363 | PASS |
| 4 | crosstech-impl-freecad-ifc-bridge | 357 | PASS |
| 5 | crosstech-impl-n8n-aec-pipeline | 309 | PASS |
| 6 | crosstech-impl-docker-aec-stack | 330 | PASS |
| 7 | crosstech-errors-conversion | 381 | PASS |
| 8 | crosstech-errors-coordinate-mismatch | 425 | PASS |
| 9 | crosstech-agents-aec-orchestrator | 328 | PASS |

### Validatie
- `docs/validation/validation-report.md` — alle 9 PASS

## Volgende stappen

### Optie A: Publiceer Wave A nu (B7)
1. Schrijf README.md met installatie-instructies en skill tabel
2. Update INDEX.md met statussen (9 DONE, 6 DEFERRED)
3. Update CHANGELOG.md
4. Push naar GitHub remote
5. Maak social preview banner
6. Tag v0.9.0 (Wave A release)
7. Stel GitHub topics in

### Optie B: Wave B skills (wanneer packages klaar zijn)
6 skills wachten op dependent packages:
- impl/ifc-to-webifc — ThatOpen package
- impl/ifc-to-threejs — Three.js package
- impl/speckle-blender — Speckle package
- impl/speckle-revit — Speckle package
- impl/qgis-bim-georef — QGIS package
- impl/bim-web-viewer — ThatOpen + Three.js packages

## Hoe te starten
```
Lees ROADMAP.md. We zijn klaar met Wave A (9 skills, gevalideerd).
Volgende stap: B7 publicatie OF Wave B skills als dependent packages klaar zijn.
```
