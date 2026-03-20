# Cross-Tech AEC Skill Package — DECISIONS

## Architectural Decisions

### D-001: Research-First Methodology
**Date:** 2026-03-20
**Decision:** Use the proven 7-phase research-first methodology from Blender, ERPNext, and Draw.io packages.
**Rationale:** This approach produces higher-quality, deterministic skills because research precedes creation. Integration skills require even MORE research than single-technology skills — both sides of every boundary must be deeply understood.

### D-002: Build LAST — Depends on All Other AEC Packages
**Date:** 2026-03-20
**Decision:** This package should be the LAST AEC skill package to be fully developed. It depends on all other AEC packages being stable.
**Rationale:** Cross-technology integration skills can only be accurate when the individual technology skills they bridge are complete. Building integration skills against incomplete or changing single-technology packages leads to rework.

### D-003: ~15 Skills Preliminary Count
**Date:** 2026-03-20
**Decision:** Target ~15 skills across 4 categories (core: 2, impl: 10, errors: 2, agents: 1).
**Rationale:** Based on initial technology boundary analysis. Covers the most critical integration points between 8 AEC technologies. Will be refined in Phase B3 after deep research.

### D-004: Claude-First
**Date:** 2026-03-20
**Decision:** Focus on the Claude ecosystem. No explicit agentskills.io compliance.
**Rationale:** Claude-specific features (allowed-tools, progressive disclosure) can be used freely without cross-platform constraints. Same decision as all other skill packages in the ecosystem.

### D-005: Each Skill Covers Exactly ONE Technology Boundary
**Date:** 2026-03-20
**Decision:** Every implementation skill covers exactly one technology boundary (A↔B). Multi-hop scenarios (A→B→C) are handled by the orchestrator agent, not by individual skills.
**Rationale:** Single-boundary skills are more maintainable, testable, and composable. The orchestrator agent chains them for complex pipelines. This prevents combinatorial explosion of skills.

### D-006: Two-Wave Build Strategy
**Date:** 2026-03-20
**Decision:** Split skill creation into Wave A (9 skills buildable now) and Wave B (6 skills deferred until dependent packages are complete). Wave A covers: core (2), automation impl (4), errors (2), agent (1). Wave B covers: web visualization impl (3), transport impl (3).
**Rationale:** 4 of 8 dependent packages (Speckle, QGIS, Three.js, ThatOpen) are empty skeleton repos. Building integration skills against incomplete packages leads to rework (per D-002). The 9 Wave A skills have all dependencies satisfied and can be written to full quality now.

### D-007: FreeCAD Skill is More Self-Contained
**Date:** 2026-03-20
**Decision:** crosstech-impl-freecad-ifc-bridge will be more self-contained than other impl skills because there is no dedicated FreeCAD skill package. It must document both sides of the FreeCAD↔IFC boundary without relying on a companion package.
**Rationale:** No FreeCAD-Claude-Skill-Package exists. The IfcOpenShell side is well-covered by the Blender-Bonsai package, but the FreeCAD side must be fully documented within this cross-tech skill.
