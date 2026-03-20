# Cross-Tech AEC Skill Package — SOURCES

## Primary Sources: Dependent AEC Skill Packages

These are the PRIMARY sources for this cross-technology package. Each contains deep research and skills for one technology.

| Package | Repository | Technologies | Status |
|---------|-----------|-------------|--------|
| Blender/Bonsai/IfcOpenShell | Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package | Blender 4.x, Bonsai BIM, IfcOpenShell, Sverchok | Reference |
| Speckle | Speckle-Claude-Skill-Package | Speckle Server, Speckle Connectors, SpecklePy | Reference |
| QGIS | QGIS-Claude-Skill-Package | QGIS 3.x, PyQGIS, GDAL/OGR | Reference |
| Three.js | Three.js-Claude-Skill-Package | Three.js r160+, WebGL, WebXR | Reference |
| ThatOpen | ThatOpen-Claude-Skill-Package | @thatopen/components, web-ifc, IFC.js | Reference |
| ERPNext | ERPNext_Anthropic_Claude_Development_Skill_Package | ERPNext v15, Frappe Framework | Reference |
| n8n | n8n-Claude-Skill-Package | n8n workflow automation | Reference |
| Docker | Docker-Claude-Skill-Package | Docker, Docker Compose | Reference |
| FreeCAD | _(no dedicated package yet)_ | FreeCAD 0.21+, FreeCAD IFC | Reference |

## Standards

| Standard | URL | Purpose | Last Verified |
|----------|-----|---------|---------------|
| IFC4 (ISO 16739-1:2018) | standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/ | Building data model schema | 2026-03-20 |
| IFC4X3 (ISO 16739-1:2024) | ifc43-docs.standards.buildingsmart.org/ | Infrastructure extension | 2026-03-20 |
| EPSG Registry | epsg.org | Coordinate Reference System codes | 2026-03-20 |
| buildingSMART Data Dictionary (bSDD) | bsdd.buildingsmart.org | Property definitions, classifications | 2026-03-20 |
| OGC Standards | ogc.org/standards | Geospatial interoperability (WMS, WFS, CityGML) | 2026-03-20 |
| CityJSON | cityjson.org | JSON-based 3D city model format | 2026-03-20 |

## Technology Documentation

| Source | URL | Purpose | Last Verified |
|--------|-----|---------|---------------|
| IfcOpenShell Documentation | ifcopenshell.org/docs | Python IFC parsing/writing API | 2026-03-20 |
| web-ifc Documentation | github.com/ThatOpen/engine_web-ifc | WebAssembly IFC parser | 2026-03-20 |
| Speckle Developer Docs | speckle.guide | Speckle server + connector APIs | 2026-03-20 |
| QGIS Documentation | docs.qgis.org | QGIS user + developer docs | 2026-03-20 |
| Three.js Documentation | threejs.org/docs | Three.js API reference | 2026-03-20 |
| Blender Python API | docs.blender.org/api/current/ | Blender scripting reference | 2026-03-20 |
| FreeCAD Python API | wiki.freecad.org/Python_scripting_tutorial | FreeCAD scripting | 2026-03-20 |
| ERPNext Developer Docs | frappeframework.com/docs | Frappe/ERPNext API | 2026-03-20 |
| n8n Documentation | docs.n8n.io | n8n workflow reference | 2026-03-20 |
| Docker Documentation | docs.docker.com | Container orchestration | 2026-03-20 |

## Coordinate Systems & Georeferencing

| Source | URL | Purpose | Last Verified |
|--------|-----|---------|---------------|
| PROJ Documentation | proj.org/en/stable/ | Coordinate transformation library | 2026-03-20 |
| pyproj Documentation | pyproj4.github.io/pyproj/stable/ | Python CRS transformations | 2026-03-20 |
| IFC Georeferencing Guide | buildingsmart.org/standards/bsi-standards/georeferencing/ | IFC map conversion, true north | 2026-03-20 |
| GDAL Documentation | gdal.org | Geospatial data abstraction | 2026-03-20 |

## Anthropic Skills Ecosystem

| Source | URL | Purpose | Last Verified |
|--------|-----|---------|---------------|
| Claude Skills Hub | claude.com/skills | Official skill marketplace | 2026-03-20 |
| Complete Guide to Building Skills | resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf | Official guide | 2026-03-20 |
| Anthropic Skills Repo | github.com/anthropics/skills | Official skills repo | 2026-03-20 |
| Claude Code Skills Docs | code.claude.com/docs/en/skills | Skill development details | 2026-03-20 |

## Source Rules
- ALWAYS prefer primary sources (dependent skill packages) over external documentation
- ALWAYS verify integration patterns against BOTH technology sides
- NEVER rely on unofficial tutorials without cross-checking against official docs
- ALWAYS cite specific versions when referencing technology capabilities
- ALWAYS verify CRS codes against the EPSG registry
