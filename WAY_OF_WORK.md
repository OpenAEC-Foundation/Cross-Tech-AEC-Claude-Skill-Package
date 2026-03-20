# Cross-Tech AEC Skill Package — WAY OF WORK

## 7-Phase Research-First Methodology

This methodology is proven across the Blender (73 skills), ERPNext (28 skills), Draw.io (22 skills), and other packages.

---

### Phase 1: Raw Masterplan
**Goal:** Define scope and preliminary skill inventory.
- Map ALL technology boundaries between AEC tools
- Identify which boundaries cause the most AI failures
- Create preliminary skill list with categories
- Define dependencies between skills AND between dependent packages
- Output: `docs/masterplan/raw-masterplan.md`

### Phase 2: Deep Research (Vooronderzoek)
**Goal:** Build comprehensive knowledge base before writing any skills.
- Minimum 2000 words per research document
- Research BOTH sides of each technology boundary
- Research documents:
  - `vooronderzoek-ifc-ecosystem.md` — IFC4/IFC4X3, IfcOpenShell, web-ifc, FreeCAD IFC
  - `vooronderzoek-bim-gis.md` — BIM↔GIS integration, coordinate systems, EPSG codes
  - `vooronderzoek-speckle-transport.md` — Speckle connectors for Blender, Revit, QGIS
  - `vooronderzoek-web-visualization.md` — Three.js + web-ifc + IFC.js viewer pipelines
  - `vooronderzoek-aec-automation.md` — n8n pipelines, Docker stacks, ERPNext costing
- Output: `docs/research/vooronderzoek-*.md`

### Phase 3: Masterplan Refinement
**Goal:** Finalize skill inventory based on research findings.
- Review research against preliminary skill list
- Merge/split/add skills based on findings
- Define per-skill specification: category, scope, both technology sides, sources, dependencies, complexity
- Write execution prompts for each skill
- Output: definitive `docs/masterplan/masterplan.md`

### Phase 4: Topic-Specific Research
**Goal:** Deep dive into each skill's specific domain.
- One research document per skill (or per logical group)
- Focus on exact API patterns, data format transformations, coordinate conversions
- Document anti-patterns and common integration mistakes
- Output: `docs/research/topic-research/{skill-name}-research.md`

### Phase 5: Skill Creation
**Goal:** Write all skills in quality-gated batches.
- 3 agents per batch, parallel execution
- Batch order follows dependency graph:
  1. Core skills (foundation, no dependencies)
  2. IFC-related impl skills (depends on core)
  3. Speckle + GIS impl skills (depends on core)
  4. Web + automation impl skills (depends on core)
  5. Error skills + agent skill (depends on all impl skills)
- Quality gate between every batch: run validation, review output

### Phase 6: Validation
**Goal:** Verify all skills meet quality requirements.
- Structural: frontmatter, line count, references
- Content: deterministic language, English-only
- Boundary coverage: BOTH sides documented for every skill
- Cross-reference: verify skill dependencies are correct
- Output: `docs/validation/validation-report.md`

### Phase 7: Publication
**Goal:** Package and publish the skill package.
- Finalize INDEX.md with complete skill catalog
- Write README.md with installation instructions
- Update CHANGELOG.md
- Tag release v1.0.0
- Update ROADMAP.md to 100% complete

---

## Skill Structure

```
skills/source/crosstech-{category}/{crosstech-{category}-{topic}}/
├── SKILL.md              # Main skill (< 500 lines, YAML frontmatter)
└── references/
    ├── methods.md         # API methods and data format mappings
    ├── examples.md        # Working integration examples
    └── anti-patterns.md   # What NOT to do at technology boundaries
```

### SKILL.md Format
```yaml
---
name: crosstech-{category}-{topic}
description: >
  Use when [specific trigger scenario — what the user is doing or asking].
  Prevents the [common mistake / anti-pattern this skill guards against].
  Covers [Technology A] ↔ [Technology B] boundary.
  Keywords: [comma-separated technical terms users might type in their prompt].
---

## Quick Reference
[One-paragraph overview of what this boundary skill bridges]

## Technology Boundary
### Side A: [Technology A]
[Key concepts, data formats, API surface]
### Side B: [Technology B]
[Key concepts, data formats, API surface]
### The Bridge
[How data/models/coordinates flow from A to B and back]

## Critical Rules
- ALWAYS {do X} — {reason}
- NEVER {do Y} — {reason}

## Decision Tree
[When to use what approach for this boundary]

## Essential Patterns
[Core integration patterns with examples]

## Common Operations
[Step-by-step workflows for crossing this boundary]

## Reference Links
[Links to official documentation for BOTH technologies]
```

---

## Quality Gates

Between every phase:
1. ROADMAP.md updated with current status
2. LESSONS.md updated with new learnings
3. DECISIONS.md updated with new decisions
4. All changes committed to git
5. BOTH sides of every technology boundary documented
