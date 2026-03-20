# Cross-Technology AEC Integration — Claude Skill Package

> **15 deterministic skills** for bridging AEC technology boundaries with Claude.
> Each skill covers exactly ONE technology boundary. English-only. ALWAYS/NEVER deterministic language.

[![Skill Quality](https://github.com/OpenAEC-Foundation/Cross-Tech-AEC-Claude-Skill-Package/actions/workflows/quality.yml/badge.svg)](https://github.com/OpenAEC-Foundation/Cross-Tech-AEC-Claude-Skill-Package/actions)

---

## Why This Package?

**AI fails most at technology boundaries.** When you ask Claude to move data from IFC to ERPNext, transform BIM coordinates to GIS, or containerize an AEC tool stack — it needs to know BOTH sides of the boundary. This package provides that knowledge.

Each skill documents:
- **Side A** — the source technology (APIs, data formats, conventions)
- **Side B** — the target technology
- **The Bridge** — how data transforms between them, and what gets lost

---

## Skills (15 skills)

### Core (2 skills) — Foundation knowledge

| Skill | Boundary | Lines |
|-------|----------|-------|
| `crosstech-core-ifc-schema-bridge` | IFC ↔ All formats | 266 |
| `crosstech-core-coordinate-systems` | BIM local ↔ GIS global | 289 |

### Implementation (10 skills) — Technology boundary bridges

| Skill | Boundary | Lines |
|-------|----------|-------|
| `crosstech-impl-ifc-to-webifc` | IfcOpenShell ↔ web-ifc | 349 |
| `crosstech-impl-ifc-to-threejs` | IFC ↔ Three.js | 463 |
| `crosstech-impl-speckle-blender` | Speckle ↔ Blender | 374 |
| `crosstech-impl-speckle-revit` | Speckle ↔ Revit | 293 |
| `crosstech-impl-qgis-bim-georef` | QGIS ↔ BIM/IFC | 435 |
| `crosstech-impl-ifc-erpnext-costing` | IFC ↔ ERPNext | 363 |
| `crosstech-impl-bim-web-viewer` | BIM/IFC ↔ Web browser | 363 |
| `crosstech-impl-freecad-ifc-bridge` | FreeCAD ↔ IFC | 357 |
| `crosstech-impl-n8n-aec-pipeline` | n8n ↔ AEC tools | 309 |
| `crosstech-impl-docker-aec-stack` | Docker ↔ AEC services | 330 |

### Errors (2 skills) — Cross-technology error diagnosis

| Skill | Boundary | Lines |
|-------|----------|-------|
| `crosstech-errors-conversion` | Any format ↔ Any format | 381 |
| `crosstech-errors-coordinate-mismatch` | BIM ↔ GIS coordinates | 425 |

### Agents (1 skill) — Pipeline orchestration

| Skill | Boundary | Lines |
|-------|----------|-------|
| `crosstech-agents-aec-orchestrator` | Multi-hop pipelines | 328 |

---

## Installation

### Option 1: Clone directly

```bash
git clone https://github.com/OpenAEC-Foundation/Cross-Tech-AEC-Claude-Skill-Package.git
```

Then add to your project's `.claude/settings.json`:

```json
{
  "skills": {
    "sources": [
      "/path/to/Cross-Tech-AEC-Claude-Skill-Package/skills/source"
    ]
  }
}
```

### Option 2: Use alongside other AEC packages

This package is designed to work WITH other AEC skill packages:

| Package | Skills | Purpose |
|---------|--------|---------|
| [Blender/Bonsai/IfcOpenShell](https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package) | 73 | 3D modeling + IFC |
| [ERPNext](https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package) | 28 | ERP/costing |
| [n8n](https://github.com/OpenAEC-Foundation/n8n-Claude-Skill-Package) | 21 | Workflow automation |
| [Docker](https://github.com/OpenAEC-Foundation/Docker-Claude-Skill-Package) | — | Containerization |
| **This package** | **15** | **Bridges between all of the above** |

---

## Technology Coverage

```
                 Blender  IfcOpenShell  Speckle  QGIS  Three.js  web-ifc  FreeCAD  ERPNext  n8n  Docker
Blender            —         (pkg)       A        —      —         —        —        —       —     —
IfcOpenShell      (pkg)       —          —        —      —         A        A        A       —     A
Speckle            A          —          —        —      —         —        —        —       A     A
QGIS               —         A          —        —      —         —        —        —       —     A
Three.js           —         —          —        —      —         A        —        —       —     —
web-ifc            —         A          —        —      A         —        —        —       —     A
FreeCAD            —         A          —        —      —         —        —        —       —     —
ERPNext            —         A          —        —      —         —        —        —       A     A
n8n                —         —          A        —      —         —        —        A       —     A
Docker             —         A          A        A      —         A        —        A       A     —

A = available skill
(pkg) = covered by single-technology package
```

---

## Quality Standards

Every skill in this package:
- Is under 500 lines (SKILL.md)
- Uses ALWAYS/NEVER deterministic language
- Documents BOTH sides of the technology boundary
- Includes `references/methods.md`, `examples.md`, and `anti-patterns.md`
- Specifies exact version numbers for all technologies
- Documents data loss at each boundary

---

## Project Structure

```
Cross-Tech-AEC-Claude-Skill-Package/
├── CLAUDE.md                    # Master configuration
├── ROADMAP.md                   # Project status
├── INDEX.md                     # Skill catalog
├── docs/
│   ├── masterplan/              # Build plans
│   ├── research/                # Deep research (~12,800 words)
│   └── validation/              # Quality reports
└── skills/source/
    ├── crosstech-core/          # 2 foundation skills
    ├── crosstech-impl/          # 10 implementation skills
    ├── crosstech-errors/        # 2 error diagnosis skills
    └── crosstech-agents/        # 1 orchestrator skill
```

---

## Methodology

Built using the [7-phase research-first methodology](https://github.com/OpenAEC-Foundation/Skill-Package-Workflow-Template):

1. Raw Masterplan
2. Deep Research (3 vooronderzoek documents, ~12,800 words)
3. Masterplan Refinement
4. Topic-Specific Research
5. Skill Creation (4 batches, 12 parallel agents)
6. Validation (100% pass rate)
7. Publication

---

## License

MIT License — see [LICENSE](LICENSE).

## Author

[OpenAEC Foundation](https://github.com/OpenAEC-Foundation)
