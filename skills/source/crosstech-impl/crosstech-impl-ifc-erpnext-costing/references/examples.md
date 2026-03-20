# crosstech-impl-ifc-erpnext-costing: Working Examples

All examples verified against research document vooronderzoek-aec-automation.md
and official documentation:
- https://docs.ifcopenshell.org/
- https://frappeframework.com/docs/user/en/api/rest
- https://docs.erpnext.com/docs/user/manual/en/manufacturing/bill-of-materials

---

## Example 1: Complete IFC Quantity to ERPNext BOM Pipeline

This is the full end-to-end pipeline: read an IFC file, extract quantities grouped by type, create Items in ERPNext, and build a BOM.

```python
"""
IFC to ERPNext Costing Pipeline
Requires: ifcopenshell 0.8.x, requests, Python 3.10+
"""
import ifcopenshell
import ifcopenshell.util.element
import ifcopenshell.util.unit
import ifcopenshell.util.classification
import requests
from collections import defaultdict

# --- Configuration ---
IFC_PATH = "project.ifc"
ERP_URL = "https://erp.example.com"
HEADERS = {
    "Authorization": "token api_key:api_secret",
    "Content-Type": "application/json"
}

# IFC quantity type to ERPNext UOM mapping
QUANTITY_UOM_MAP = {
    "length": "Meter",
    "area": "Square Meter",
    "volume": "Cubic Meter",
    "count": "Nos",
    "weight": "Kg",
}

# IFC entity type to ERPNext Item Group fallback
ENTITY_GROUP_MAP = {
    "IfcWall": "Walls",
    "IfcSlab": "Slabs",
    "IfcBeam": "Beams",
    "IfcColumn": "Columns",
    "IfcDoor": "Doors",
    "IfcWindow": "Windows",
    "IfcRoof": "Roofs",
    "IfcStair": "Stairs",
    "IfcFooting": "Foundations",
    "IfcPile": "Foundations",
    "IfcCurtainWall": "Curtain Walls",
    "IfcPlate": "Plates",
    "IfcMember": "Structural Members",
    "IfcFurnishingElement": "Furniture",
}

# --- Step 1: Parse IFC and get unit scale ---
model = ifcopenshell.open(IFC_PATH)
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
print(f"Schema: {model.schema}, Unit scale: {unit_scale}")

# --- Step 2: Extract quantities grouped by element type ---
type_data = defaultdict(lambda: {
    "quantities": defaultdict(float),
    "count": 0,
    "ifc_type": "",
    "classification": None,
    "primary_uom": "Nos",
})

# Process all building elements
for element in model.by_type("IfcElement"):
    element_type = ifcopenshell.util.element.get_type(element)
    type_name = element_type.Name if element_type else f"UNTYPED_{element.is_a()}"
    ifc_class = element.is_a()

    entry = type_data[type_name]
    entry["count"] += 1
    entry["ifc_type"] = ifc_class

    # Get classification (use first element's classification for the type)
    if entry["classification"] is None:
        refs = ifcopenshell.util.classification.get_references(element)
        if refs:
            entry["classification"] = {
                "code": refs[0].Identification,
                "name": refs[0].Name,
            }

    # Extract quantities with unit normalization
    qtos = ifcopenshell.util.element.get_psets(element, qtos_only=True)
    for qto_name, quantities in qtos.items():
        for qty_name, qty_value in quantities.items():
            if qty_name == "id":  # ALWAYS skip the IfcOpenShell entity ID
                continue
            if qty_value is None or qty_value == 0:
                continue

            # Normalize based on quantity name conventions
            if "Volume" in qty_name:
                entry["quantities"][qty_name] += qty_value * unit_scale**3
                entry["primary_uom"] = "Cubic Meter"
            elif "Area" in qty_name or "Surface" in qty_name:
                entry["quantities"][qty_name] += qty_value * unit_scale**2
                entry["primary_uom"] = "Square Meter"
            elif "Length" in qty_name or "Height" in qty_name or "Width" in qty_name:
                entry["quantities"][qty_name] += qty_value * unit_scale
            elif "Weight" in qty_name:
                entry["quantities"][qty_name] += qty_value  # Weight has separate unit
                entry["primary_uom"] = "Kg"
            elif "Count" in qty_name:
                entry["quantities"][qty_name] += qty_value
                entry["primary_uom"] = "Nos"
            else:
                entry["quantities"][qty_name] += qty_value * unit_scale

# --- Step 3: Create Items in ERPNext ---
def ensure_item_group(group_name: str) -> None:
    """Create Item Group if it does not exist."""
    check = requests.get(
        f"{ERP_URL}/api/resource/Item Group/{group_name}",
        headers=HEADERS
    )
    if check.status_code == 200:
        return
    requests.post(
        f"{ERP_URL}/api/resource/Item Group",
        json={
            "doctype": "Item Group",
            "item_group_name": group_name,
            "parent_item_group": "All Item Groups"
        },
        headers=HEADERS
    ).raise_for_status()


def sanitize_item_code(name: str) -> str:
    """Convert IFC type name to valid ERPNext item_code."""
    return name.replace(" ", "-").replace(":", "-").replace("/", "-").upper()[:140]


created_items = []
for type_name, data in type_data.items():
    # Determine Item Group
    if data["classification"]:
        group_name = data["classification"]["name"]
    else:
        group_name = ENTITY_GROUP_MAP.get(data["ifc_type"], "Other BIM Elements")

    ensure_item_group(group_name)

    item_code = sanitize_item_code(type_name)

    # Check if item exists
    check = requests.get(
        f"{ERP_URL}/api/resource/Item/{item_code}",
        headers=HEADERS
    )
    if check.status_code != 200:
        item_data = {
            "doctype": "Item",
            "item_code": item_code,
            "item_name": type_name,
            "item_group": group_name,
            "stock_uom": data["primary_uom"],
            "description": f"IFC {data['ifc_type']} — {type_name} ({data['count']} instances)"
        }
        response = requests.post(
            f"{ERP_URL}/api/resource/Item",
            json=item_data,
            headers=HEADERS
        )
        response.raise_for_status()
        print(f"Created Item: {item_code}")

    # Determine the primary quantity for BOM
    quantities = data["quantities"]
    if "GrossVolume" in quantities:
        primary_qty = quantities["GrossVolume"]
        uom = "Cubic Meter"
    elif "GrossSideArea" in quantities or "Area" in quantities:
        primary_qty = quantities.get("GrossSideArea", quantities.get("Area", 0))
        uom = "Square Meter"
    elif "GrossWeight" in quantities:
        primary_qty = quantities["GrossWeight"]
        uom = "Kg"
    else:
        primary_qty = data["count"]
        uom = "Nos"

    if primary_qty > 0:
        created_items.append({
            "item_code": item_code,
            "qty": round(primary_qty, 4),
            "uom": uom,
        })

# --- Step 4: Create BOM ---
project_name = model.by_type("IfcProject")[0].Name or "IFC Project"
building_item_code = sanitize_item_code(project_name)

# Create the parent building item
check = requests.get(
    f"{ERP_URL}/api/resource/Item/{building_item_code}",
    headers=HEADERS
)
if check.status_code != 200:
    requests.post(
        f"{ERP_URL}/api/resource/Item",
        json={
            "doctype": "Item",
            "item_code": building_item_code,
            "item_name": project_name,
            "item_group": "All Item Groups",
            "stock_uom": "Nos",
            "is_stock_item": 0,
        },
        headers=HEADERS
    ).raise_for_status()

bom_data = {
    "doctype": "BOM",
    "item": building_item_code,
    "quantity": 1,
    "items": created_items,
    "rm_cost_as_per": "Valuation Rate",
}

response = requests.post(
    f"{ERP_URL}/api/resource/BOM",
    json=bom_data,
    headers=HEADERS
)
response.raise_for_status()
bom_name = response.json()["data"]["name"]
print(f"Created BOM: {bom_name}")

# --- Step 5: Submit BOM ---
requests.put(
    f"{ERP_URL}/api/resource/BOM/{bom_name}",
    json={"docstatus": 1},
    headers=HEADERS
).raise_for_status()
print(f"Submitted BOM: {bom_name}")
```

---

## Example 2: Extract Quantities for a Single Storey

```python
"""Extract quantities for elements on a specific building storey."""
import ifcopenshell
import ifcopenshell.util.element
import ifcopenshell.util.unit

model = ifcopenshell.open("project.ifc")
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

# Find the target storey
target_storey_name = "Level 01"
storey = None
for s in model.by_type("IfcBuildingStorey"):
    if s.Name == target_storey_name:
        storey = s
        break

if storey is None:
    raise ValueError(f"Storey '{target_storey_name}' not found in IFC model")

# Get all elements contained in this storey
storey_elements = []
for rel in storey.ContainsElements:
    storey_elements.extend(rel.RelatedElements)

print(f"Storey '{target_storey_name}' contains {len(storey_elements)} elements")

# Extract quantities per element
for element in storey_elements:
    qtos = ifcopenshell.util.element.get_psets(element, qtos_only=True)
    if not qtos:
        continue

    element_type = ifcopenshell.util.element.get_type(element)
    type_name = element_type.Name if element_type else element.Name or "Unnamed"

    print(f"\n{element.is_a()} — {type_name} (#{element.id()})")
    for qto_name, quantities in qtos.items():
        for qty_name, qty_value in quantities.items():
            if qty_name == "id" or qty_value is None:
                continue
            print(f"  {qto_name}.{qty_name} = {qty_value}")
```

---

## Example 3: Multi-Level BOM (Building → Storeys → Elements)

```python
"""Create hierarchical BOMs: Building BOM references Storey BOMs."""
import ifcopenshell
import ifcopenshell.util.element
import ifcopenshell.util.unit
import requests
from collections import defaultdict

model = ifcopenshell.open("project.ifc")
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

ERP_URL = "https://erp.example.com"
HEADERS = {
    "Authorization": "token api_key:api_secret",
    "Content-Type": "application/json"
}

storey_bom_names = []

for storey in model.by_type("IfcBuildingStorey"):
    storey_item_code = f"STOREY-{storey.Name}".replace(" ", "-").upper()[:140]

    # Create storey-level Item
    requests.post(
        f"{ERP_URL}/api/resource/Item",
        json={
            "doctype": "Item",
            "item_code": storey_item_code,
            "item_name": storey.Name,
            "item_group": "Building Storeys",
            "stock_uom": "Nos",
            "is_stock_item": 0,
        },
        headers=HEADERS
    )

    # Collect element quantities for this storey
    bom_items = []
    for rel in storey.ContainsElements:
        for element in rel.RelatedElements:
            element_type = ifcopenshell.util.element.get_type(element)
            if not element_type:
                continue

            item_code = element_type.Name.replace(" ", "-").replace(":", "-").upper()[:140]
            qtos = ifcopenshell.util.element.get_psets(element, qtos_only=True)

            # Find the primary quantity
            for qto_name, quantities in qtos.items():
                volume = quantities.get("GrossVolume")
                if volume and volume > 0:
                    bom_items.append({
                        "item_code": item_code,
                        "qty": round(volume * unit_scale**3, 4),
                        "uom": "Cubic Meter",
                    })
                    break

    if not bom_items:
        continue

    # Create storey BOM
    response = requests.post(
        f"{ERP_URL}/api/resource/BOM",
        json={
            "doctype": "BOM",
            "item": storey_item_code,
            "quantity": 1,
            "items": bom_items,
            "rm_cost_as_per": "Valuation Rate",
        },
        headers=HEADERS
    )
    response.raise_for_status()
    bom_name = response.json()["data"]["name"]

    # Submit storey BOM
    requests.put(
        f"{ERP_URL}/api/resource/BOM/{bom_name}",
        json={"docstatus": 1},
        headers=HEADERS
    ).raise_for_status()

    storey_bom_names.append({
        "item_code": storey_item_code,
        "qty": 1,
        "uom": "Nos",
    })
    print(f"Created storey BOM: {bom_name}")

# Create building-level BOM referencing storey BOMs
building_name = model.by_type("IfcProject")[0].Name or "Building"
building_item_code = f"BUILDING-{building_name}".replace(" ", "-").upper()[:140]

requests.post(
    f"{ERP_URL}/api/resource/Item",
    json={
        "doctype": "Item",
        "item_code": building_item_code,
        "item_name": building_name,
        "item_group": "Buildings",
        "stock_uom": "Nos",
        "is_stock_item": 0,
    },
    headers=HEADERS
)

response = requests.post(
    f"{ERP_URL}/api/resource/BOM",
    json={
        "doctype": "BOM",
        "item": building_item_code,
        "quantity": 1,
        "items": storey_bom_names,
        "rm_cost_as_per": "Valuation Rate",
    },
    headers=HEADERS
)
response.raise_for_status()
building_bom = response.json()["data"]["name"]

requests.put(
    f"{ERP_URL}/api/resource/BOM/{building_bom}",
    json={"docstatus": 1},
    headers=HEADERS
).raise_for_status()
print(f"Created building BOM: {building_bom}")
```

---

## Example 4: Validate IFC Quantities Before ERPNext Import

```python
"""Validate that IFC quantities are complete and consistent before import."""
import ifcopenshell
import ifcopenshell.util.element
import ifcopenshell.util.unit

model = ifcopenshell.open("project.ifc")
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)

issues = []

for element in model.by_type("IfcElement"):
    element_id = f"{element.is_a()} #{element.id()} ({element.Name or 'unnamed'})"

    # Check 1: Does the element have a type?
    element_type = ifcopenshell.util.element.get_type(element)
    if element_type is None:
        issues.append(f"NO TYPE: {element_id}")

    # Check 2: Does the element have quantity sets?
    qtos = ifcopenshell.util.element.get_psets(element, qtos_only=True)
    if not qtos:
        issues.append(f"NO QUANTITIES: {element_id}")
        continue

    # Check 3: Are all quantity values non-zero?
    for qto_name, quantities in qtos.items():
        for qty_name, qty_value in quantities.items():
            if qty_name == "id":
                continue
            if qty_value is None:
                issues.append(f"NULL QUANTITY: {element_id} → {qto_name}.{qty_name}")
            elif qty_value == 0:
                issues.append(f"ZERO QUANTITY: {element_id} → {qto_name}.{qty_name}")
            elif qty_value < 0:
                issues.append(f"NEGATIVE QUANTITY: {element_id} → {qto_name}.{qty_name} = {qty_value}")

print(f"\nValidation complete: {len(issues)} issues found")
for issue in issues:
    print(f"  - {issue}")
```
