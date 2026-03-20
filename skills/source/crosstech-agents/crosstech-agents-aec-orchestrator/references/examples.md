# crosstech-agents-aec-orchestrator — Examples Reference

## Example 1: FreeCAD → IFC → ERPNext Cost Estimate

**User request**: "I have a building model in FreeCAD. I need to generate a cost estimate in ERPNext."

**Decomposition**:
- Technologies detected: FreeCAD, IFC (implicit), ERPNext
- Boundaries: FreeCAD ↔ IFC (#4), IFC ↔ ERPNext (#3)
- Foundation skills needed: `crosstech-core-ifc-schema-bridge` (IFC involved)

**Skills loaded** (in order):
1. `crosstech-core-ifc-schema-bridge` — IFC schema reference
2. `crosstech-impl-freecad-ifc-bridge` — FreeCAD → IFC export
3. `crosstech-impl-ifc-erpnext-costing` — IFC quantities → ERPNext BOM

**Pipeline execution**:

```python
# Hop 1: Export IFC from FreeCAD (NativeIFC mode)
# Skill: crosstech-impl-freecad-ifc-bridge
import FreeCAD
import exportIFC

doc = FreeCAD.openDocument("building.FCStd")
exportIFC.export([doc.Objects], "building.ifc")

# Validation between Hop 1 and Hop 2
import ifcopenshell
model = ifcopenshell.open("building.ifc")
assert model.schema in ("IFC4", "IFC4X3"), f"Unexpected schema: {model.schema}"
assert len(model.by_type("IfcProduct")) > 0, "No products in exported IFC"

# Hop 2: Extract quantities and map to ERPNext
# Skill: crosstech-impl-ifc-erpnext-costing
import ifcopenshell.util.element as elem_util

walls = model.by_type("IfcWall")
bom_items = []
for wall in walls:
    psets = elem_util.get_psets(wall)
    qsets = {k: v for k, v in psets.items() if "Quantity" in k or "BaseQuantities" in k}

    area = qsets.get("Qto_WallBaseQuantities", {}).get("NetSideArea", 0)
    volume = qsets.get("Qto_WallBaseQuantities", {}).get("NetVolume", 0)

    if area > 0:
        bom_items.append({
            "item_code": f"WALL-{wall.Name or wall.GlobalId}",
            "qty": area,
            "uom": "Square Meter",
            "description": f"Wall: {wall.Name}"
        })

# Validation after Hop 2
assert len(bom_items) > 0, "No BOM items generated — check quantity sets"
for item in bom_items:
    assert item["qty"] > 0, f"Zero quantity for {item['item_code']}"
```

---

## Example 2: IFC Validation Pipeline with Docker + n8n

**User request**: "Set up an automated pipeline that validates uploaded IFC files and rejects invalid ones."

**Decomposition**:
- Technologies detected: IFC, Docker, n8n
- Boundaries: Docker ↔ AEC (#6), n8n ↔ AEC (#5)
- Foundation skills needed: `crosstech-core-ifc-schema-bridge` (IFC validation)

**Skills loaded** (in order):
1. `crosstech-core-ifc-schema-bridge` — IFC schema reference
2. `crosstech-impl-docker-aec-stack` — Containerized IfcOpenShell
3. `crosstech-impl-n8n-aec-pipeline` — Webhook trigger + workflow

**Pipeline execution**:

```yaml
# docker-compose.yml — IfcOpenShell validation container
# Skill: crosstech-impl-docker-aec-stack
version: "3.8"
services:
  ifc-validator:
    build:
      context: .
      dockerfile: Dockerfile.validator
    volumes:
      - ./uploads:/data/uploads
      - ./reports:/data/reports
    environment:
      - ACCEPTED_SCHEMAS=IFC4,IFC4X3
```

```python
# validate_ifc.py — runs inside Docker container
# Skill: crosstech-core-ifc-schema-bridge
import ifcopenshell
import ifcopenshell.util.element
import json
import sys

def validate(filepath: str) -> dict:
    report = {"file": filepath, "errors": [], "warnings": [], "valid": True}

    try:
        model = ifcopenshell.open(filepath)
    except Exception as e:
        report["errors"].append(f"Cannot parse IFC: {e}")
        report["valid"] = False
        return report

    # Schema check
    if model.schema not in ("IFC4", "IFC4X3"):
        report["errors"].append(f"Unsupported schema: {model.schema}")
        report["valid"] = False

    # Entity check
    products = model.by_type("IfcProduct")
    if len(products) == 0:
        report["errors"].append("No IfcProduct entities found")
        report["valid"] = False

    # Property check (sample 3 elements)
    import random
    samples = random.sample(products, min(3, len(products)))
    for elem in samples:
        psets = ifcopenshell.util.element.get_psets(elem)
        if not psets:
            report["warnings"].append(f"{elem.is_a()} #{elem.id()} has no property sets")

    return report

if __name__ == "__main__":
    result = validate(sys.argv[1])
    print(json.dumps(result, indent=2))
    sys.exit(0 if result["valid"] else 1)
```

```json
// n8n workflow definition (simplified)
// Skill: crosstech-impl-n8n-aec-pipeline
{
  "nodes": [
    {
      "name": "Webhook Trigger",
      "type": "n8n-nodes-base.webhook",
      "parameters": {
        "path": "ifc-upload",
        "httpMethod": "POST",
        "responseMode": "lastNode"
      }
    },
    {
      "name": "Save File",
      "type": "n8n-nodes-base.writeFile",
      "parameters": {
        "fileName": "/data/uploads/{{ $json.filename }}"
      }
    },
    {
      "name": "Run Validator",
      "type": "n8n-nodes-base.executeCommand",
      "parameters": {
        "command": "docker exec ifc-validator python validate_ifc.py /data/uploads/{{ $json.filename }}"
      }
    },
    {
      "name": "Check Result",
      "type": "n8n-nodes-base.if",
      "parameters": {
        "conditions": {
          "boolean": [{ "value1": "={{ $json.valid }}", "value2": true }]
        }
      }
    }
  ]
}
```

---

## Example 3: FreeCAD Model → Docker Processing → n8n Report

**User request**: "I want to export my FreeCAD model, validate it in Docker, and get an automated report via n8n."

**Decomposition**:
- Technologies detected: FreeCAD, IFC (implicit), Docker, n8n
- Boundaries: FreeCAD ↔ IFC (#4), Docker ↔ AEC (#6), n8n ↔ AEC (#5)
- Foundation skills needed: `crosstech-core-ifc-schema-bridge`
- Hop count: 3

**Skills loaded** (in order):
1. `crosstech-core-ifc-schema-bridge`
2. `crosstech-impl-freecad-ifc-bridge`
3. `crosstech-impl-docker-aec-stack`
4. `crosstech-impl-n8n-aec-pipeline`

**Pipeline with validation checkpoints**:

```python
# === HOP 1: FreeCAD → IFC ===
# Skill: crosstech-impl-freecad-ifc-bridge

import subprocess
result = subprocess.run([
    "freecadcmd", "-c",
    "import FreeCAD; import exportIFC; "
    "doc = FreeCAD.openDocument('model.FCStd'); "
    "exportIFC.export(doc.Objects, '/data/output/model.ifc')"
], capture_output=True, text=True)

assert result.returncode == 0, f"FreeCAD export failed: {result.stderr}"

# === VALIDATION CHECKPOINT 1 ===
import ifcopenshell
model = ifcopenshell.open("/data/output/model.ifc")
assert model.schema in ("IFC4", "IFC4X3")
product_count = len(model.by_type("IfcProduct"))
assert product_count > 0, "Empty IFC file"
print(f"Checkpoint 1 PASSED: {product_count} products, schema {model.schema}")

# === HOP 2: Docker Validation ===
# Skill: crosstech-impl-docker-aec-stack

result = subprocess.run([
    "docker", "exec", "ifc-validator",
    "python", "validate_ifc.py", "/data/output/model.ifc"
], capture_output=True, text=True)

import json
report = json.loads(result.stdout)

# === VALIDATION CHECKPOINT 2 ===
assert report["valid"], f"Validation failed: {report['errors']}"
print(f"Checkpoint 2 PASSED: {len(report['warnings'])} warnings")

# === HOP 3: n8n Report Generation ===
# Skill: crosstech-impl-n8n-aec-pipeline

import requests
response = requests.post("http://n8n:5678/webhook/report-generation", json={
    "file": "model.ifc",
    "validation_report": report,
    "product_count": product_count,
    "schema": model.schema
})

# === VALIDATION CHECKPOINT 3 ===
assert response.status_code == 200, f"n8n webhook failed: {response.status_code}"
print(f"Checkpoint 3 PASSED: Report generated at {response.json().get('report_url')}")
```

---

## Example 4: Handling a Deferred Skill (Wave B)

**User request**: "I want to view my IFC model in a Three.js web viewer."

**Decomposition**:
- Technologies detected: IFC, Three.js
- Boundary: IFC ↔ Three.js (#11) — **DEFERRED**

**Orchestrator response**:

```
The skill crosstech-impl-ifc-to-threejs is DEFERRED (Wave B).
It requires the Three.js-Claude-Skill-Package to be installed.

Available workaround using Wave A skills:
1. Load crosstech-core-ifc-schema-bridge — understand IFC entity mapping
2. Load crosstech-core-coordinate-systems — handle Y-up/Z-up axis swap

Manual steps required:
- Use @thatopen/components or web-ifc-three for IFC loading
- Apply Y/Z axis swap: Three.js_y = BIM_z, Three.js_z = -BIM_y
- Maintain parallel property store (Three.js scene graph does NOT carry IFC properties)
- See crosstech-core-ifc-schema-bridge data loss matrix for what is lost in Three.js
```

---

## Example 5: Error Recovery in a Multi-Hop Pipeline

**Scenario**: Hop 2 of a 3-hop pipeline fails with missing geometry.

**Error recovery sequence**:

```
Pipeline: FreeCAD → IFC → Docker validation → n8n report
Hop 2 FAILED: Docker validator reports "0 geometry representations found"

Step 1: Load crosstech-errors-conversion
  → Follow symptom-to-root-cause tree
  → Symptom: "Missing geometry after conversion"
  → Root cause candidates:
    a) FreeCAD export did not include geometry (empty shapes)
    b) IFC file has BREP geometry not supported by validator
    c) Schema mismatch in validator parser

Step 2: Diagnose
  → Check input to Hop 2 (the IFC file from Hop 1):
    model = ifcopenshell.open("model.ifc")
    for product in model.by_type("IfcProduct")[:5]:
        rep = product.Representation
        print(f"{product.Name}: {'HAS geometry' if rep else 'NO geometry'}")
  → If geometry exists in input but not in output: validator bug
  → If geometry missing in input: FreeCAD export issue

Step 3: Fix the specific failure
  → If FreeCAD export issue: re-export with explicit geometry settings
  → If validator bug: update IfcOpenShell version in Docker container

Step 4: Retry ONLY Hop 2 (do NOT restart from Hop 1)
  → Re-run Docker validation
  → Re-run validation checkpoint
  → If passed: continue to Hop 3
```
