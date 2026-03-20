---
name: crosstech-impl-ifc-erpnext-costing
description: >
  Use when extracting quantities from IFC models and creating cost estimates in ERPNext.
  Prevents the common mistake of mapping IFC quantities to wrong ERPNext fields
  or losing unit information during the IFC-to-BOM conversion.
  Covers IfcElementQuantity extraction via IfcOpenShell, ERPNext BOM structure,
  IFC classification to Item Group mapping, and the complete quantity-to-cost pipeline.
  Keywords: IFC costing, BOM, ERPNext, IfcElementQuantity, quantity takeoff,
  IfcOpenShell, Frappe API, cost estimation, IFC to ERP.
license: MIT
compatibility: "Designed for Claude Code. Requires IfcOpenShell 0.8.x and ERPNext v15."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-impl-ifc-erpnext-costing

## Quick Reference

### IFC Quantity Types (all 5)

| IFC Type | Value Unit | ERPNext UOM | Typical Use |
|----------|-----------|-------------|-------------|
| `IfcQuantityLength` | m | Meter | Beam lengths, pipe runs, cable trays |
| `IfcQuantityArea` | m2 | Square Meter | Wall surfaces, floor areas, roof areas |
| `IfcQuantityVolume` | m3 | Cubic Meter | Concrete volumes, excavation volumes |
| `IfcQuantityCount` | integer | Nos | Door count, window count, fixture count |
| `IfcQuantityWeight` | kg | Kg | Steel tonnage, rebar weight |

### Standard Quantity Sets (Qto_ prefix)

| Quantity Set | Element Types | Key Quantities |
|-------------|--------------|----------------|
| `Qto_WallBaseQuantities` | IfcWall | Length, Height, Width, GrossSideArea, NetSideArea, GrossVolume, NetVolume |
| `Qto_SlabBaseQuantities` | IfcSlab | Area, NominalThickness, GrossVolume, NetVolume, GrossWeight |
| `Qto_BeamBaseQuantities` | IfcBeam | Length, CrossSectionArea, GrossSurfaceArea, NetVolume |
| `Qto_ColumnBaseQuantities` | IfcColumn | Length, CrossSectionArea, GrossSurfaceArea, GrossVolume |
| `Qto_DoorBaseQuantities` | IfcDoor | Height, Width, Area |
| `Qto_WindowBaseQuantities` | IfcWindow | Height, Width, Area |

### Critical Warnings

**NEVER** map IFC quantities to ERPNext without checking the IFC file's unit system first. Use `ifcopenshell.util.unit.calculate_unit_scale(model)` to get the scale factor to SI units.

**NEVER** submit an ERPNext BOM without verifying that all referenced Items exist. The Frappe API returns a 400 error with no rollback if an Item is missing.

**NEVER** use `IfcQuantityVolume` for cost estimation of composite elements (e.g., layered walls). The gross volume includes ALL material layers — use material-specific quantities instead.

**NEVER** assume every IFC element has quantity sets. Many IFC files export with empty `Qto_` sets or no quantity sets at all. ALWAYS validate that quantities are non-null before creating BOM Items.

**NEVER** create duplicate Items in ERPNext from IFC instances of the same type. Group by `IfcElementType` first, then aggregate quantities across instances.

**ALWAYS** use `ifcopenshell.util.element.get_psets(element, qtos_only=True)` to extract quantities — this merges occurrence-level and type-level quantity sets automatically.

---

## Technology Boundary

### Side A: IFC Model (IfcOpenShell 0.8.x)

**Data format**: IFC-SPF (.ifc) or IFC-XML (.ifcXML), parsed into `ifcopenshell.file` object.

**API surface for quantity extraction**:

| Function | Purpose |
|----------|---------|
| `ifcopenshell.open(path)` | Load IFC file into memory |
| `model.by_type("IfcWall")` | Get all elements of a type |
| `ifcopenshell.util.element.get_psets(elem, qtos_only=True)` | Extract quantity sets |
| `ifcopenshell.util.element.get_psets(elem, psets_only=True)` | Extract property sets |
| `ifcopenshell.util.element.get_type(elem)` | Get the IfcElementType |
| `ifcopenshell.util.classification.get_references(elem)` | Get classification codes |
| `ifcopenshell.util.unit.calculate_unit_scale(model)` | Get SI unit scale factor |
| `model.schema` | Check schema version (IFC2X3, IFC4, IFC4X3) |

**Data available**: geometry, properties, quantities, classifications, materials, spatial hierarchy, relationships.

### Side B: ERPNext ERP (Frappe REST API, ERPNext v15)

**Data format**: JSON via REST API at `/api/resource/:doctype`.

**Core DocTypes for costing**:

| DocType | Purpose | Key Fields |
|---------|---------|------------|
| `Item` | Master record for materials/products | `item_code`, `item_name`, `item_group`, `stock_uom`, `description` |
| `BOM` | Bill of Materials (assembly recipe) | `item`, `items` (child table), `rm_cost_as_per` |
| `BOM Item` | Component row within a BOM | `item_code`, `qty`, `uom`, `rate` |
| `Project` | Top-level project container | `project_name`, `company` |
| `Item Group` | Hierarchical item categorization | `name`, `parent_item_group` |

**Authentication**: Token-based header: `Authorization: token api_key:api_secret`.

**Data available**: pricing, valuation rates, supplier info, warehouse locations, serial/batch tracking, cost centers.

### The Bridge: Quantity Extraction to BOM Creation Pipeline

```
IFC Model (.ifc)
    │
    ▼
[1] Parse with IfcOpenShell
    │  ifcopenshell.open("project.ifc")
    │  Check units: ifcopenshell.util.unit.calculate_unit_scale(model)
    │
    ▼
[2] Extract element types + quantities
    │  Group elements by IfcElementType
    │  For each type: aggregate quantities across all instances
    │  Map classification codes to ERPNext Item Groups
    │
    ▼
[3] Create Items in ERPNext
    │  POST /api/resource/Item for each unique element type
    │  Map IFC unit → ERPNext UOM
    │  Map IFC classification → Item Group
    │
    ▼
[4] Create BOM in ERPNext
    │  POST /api/resource/BOM with aggregated quantities
    │  Reference Items created in step 3
    │
    ▼
[5] Submit BOM
    PUT /api/resource/BOM/:name with docstatus=1
```

**Directionality**: IFC → ERPNext is the primary flow. ERPNext → IFC is NOT supported (ERPNext has no geometry model, no spatial relationships, no IFC schema awareness).

---

## Critical Rules

1. **ALWAYS** check the IFC unit system before extracting quantities. Multiply all quantity values by the unit scale factor to normalize to SI units.
2. **ALWAYS** group IFC elements by their `IfcElementType` before creating ERPNext Items. One Item per type, NOT one Item per element instance.
3. **ALWAYS** verify that an Item exists in ERPNext (GET request) before referencing it in a BOM. Create missing Items first.
4. **ALWAYS** use the `qtos_only=True` parameter when extracting quantities to avoid mixing property values with quantity values.
5. **ALWAYS** map IFC units to ERPNext UOM names exactly: `m` → `Meter`, `m2` → `Square Meter`, `m3` → `Cubic Meter`, `kg` → `Kg`.
6. **ALWAYS** handle the case where quantity sets are empty or missing — skip elements with no quantities instead of creating zero-quantity BOM Items.
7. **NEVER** submit a BOM (docstatus=1) until all Items and quantities are verified. Submitted BOMs are immutable in ERPNext.
8. **NEVER** use hardcoded unit conversions. Use the IFC file's `IfcUnitAssignment` via `calculate_unit_scale()`.
9. **NEVER** flatten IFC spatial hierarchy for cost tracking without mapping storeys to ERPNext Cost Centers first.
10. **NEVER** send more than 50 API requests per second to ERPNext — use batch creation or rate limiting.

---

## Decision Tree

```
START: You have an IFC file and need cost estimates in ERPNext
│
├── Q1: Does the IFC file contain quantity sets (Qto_ prefixed)?
│   ├── YES → Use existing quantities. Proceed to Q2.
│   └── NO → Quantities must be calculated from geometry.
│       └── Use IfcOpenShell geometry engine to compute volumes/areas.
│           model.by_type("IfcWall") → create_shape() → compute from mesh.
│           WARNING: Geometry-derived quantities are approximations.
│
├── Q2: What unit system does the IFC file use?
│   ├── Check: ifcopenshell.util.unit.calculate_unit_scale(model)
│   ├── Returns 1.0 → File uses meters (SI). No conversion needed.
│   ├── Returns 0.001 → File uses millimeters. Multiply all lengths by 0.001.
│   └── Returns other → Apply scale factor to ALL extracted quantities.
│
├── Q3: Does the IFC file have classification references?
│   ├── YES → Map classification codes to ERPNext Item Groups.
│   │   ├── Uniclass → Use Identification field (e.g., "Ss_25_10_30")
│   │   └── OmniClass → Use Identification field (e.g., "23-13 21 00")
│   └── NO → Use IFC element type names as Item Group fallback.
│       └── Group by is_a() result: "IfcWall" → "Walls", "IfcSlab" → "Slabs"
│
├── Q4: Single-level or multi-level BOM?
│   ├── SINGLE → One BOM per building storey with all element quantities.
│   └── MULTI → Parent BOM (building) → child BOMs (storeys) → element Items.
│       └── Create hierarchical BOMs: Building BOM references Storey BOMs.
│
└── Q5: Cost source in ERPNext?
    ├── Valuation Rate → Uses average stock valuation. Set rm_cost_as_per="Valuation Rate".
    ├── Last Purchase Rate → Uses most recent purchase price.
    └── Price List → Uses a specific price list. Specify price_list field.
```

---

## Essential Patterns

### Pattern 1: Extract Quantities with Unit Normalization

```python
import ifcopenshell
import ifcopenshell.util.element
import ifcopenshell.util.unit

model = ifcopenshell.open("project.ifc")

# ALWAYS get unit scale first
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
# unit_scale converts file units to meters (e.g., 0.001 for mm files)

for wall in model.by_type("IfcWall"):
    qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
    qto = qtos.get("Qto_WallBaseQuantities", {})

    # Apply unit scale to length/area/volume quantities
    length = qto.get("Length", 0) * unit_scale          # now in meters
    area = qto.get("GrossSideArea", 0) * unit_scale**2  # now in m2
    volume = qto.get("GrossVolume", 0) * unit_scale**3  # now in m3
    # Count and Weight do NOT need unit_scale conversion
```

### Pattern 2: Group by Element Type for Item Creation

```python
from collections import defaultdict

type_quantities = defaultdict(lambda: {"count": 0, "volume": 0.0, "area": 0.0})

for wall in model.by_type("IfcWall"):
    wall_type = ifcopenshell.util.element.get_type(wall)
    type_name = wall_type.Name if wall_type else "UNTYPED_WALL"

    qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
    qto = qtos.get("Qto_WallBaseQuantities", {})

    type_quantities[type_name]["count"] += 1
    type_quantities[type_name]["volume"] += qto.get("GrossVolume", 0) * unit_scale**3
    type_quantities[type_name]["area"] += qto.get("GrossSideArea", 0) * unit_scale**2

# Result: {"Concrete Wall 200mm": {"count": 12, "volume": 45.6, "area": 228.0}, ...}
```

### Pattern 3: Create ERPNext Items from IFC Types

```python
import requests

ERP_URL = "https://erp.example.com"
HEADERS = {
    "Authorization": "token api_key:api_secret",
    "Content-Type": "application/json"
}

IFC_UNIT_TO_ERP_UOM = {
    "length": "Meter",
    "area": "Square Meter",
    "volume": "Cubic Meter",
    "count": "Nos",
    "weight": "Kg",
}

def create_item(item_code: str, item_name: str, group: str, uom: str) -> dict:
    """Create an ERPNext Item. Returns the created item data."""
    # ALWAYS check if item exists first
    check = requests.get(
        f"{ERP_URL}/api/resource/Item/{item_code}",
        headers=HEADERS
    )
    if check.status_code == 200:
        return check.json()["data"]  # Item already exists

    item_data = {
        "doctype": "Item",
        "item_code": item_code,
        "item_name": item_name,
        "item_group": group,
        "stock_uom": uom,
        "description": f"Auto-created from IFC model ({item_name})"
    }
    response = requests.post(
        f"{ERP_URL}/api/resource/Item",
        json=item_data,
        headers=HEADERS
    )
    response.raise_for_status()
    return response.json()["data"]
```

---

## Common Operations

### IFC Unit to ERPNext UOM Mapping

| IFC Unit (from IfcUnitAssignment) | unit_scale Result | ERPNext UOM | Conversion |
|----------------------------------|-------------------|-------------|------------|
| METRE | 1.0 | Meter | Direct |
| MILLIMETRE | 0.001 | Meter | value * 0.001 |
| SQUARE_METRE | 1.0 | Square Meter | Direct |
| CUBIC_METRE | 1.0 | Cubic Meter | Direct |
| KILOGRAM | 1.0 | Kg | Direct |
| POUND | 0.453592 | Kg | value * 0.453592 |

### IFC Classification to ERPNext Item Group Mapping

| IFC Source | Field | ERPNext Target | Example |
|-----------|-------|---------------|---------|
| `IfcClassificationReference.Identification` | Uniclass code | `Item.item_group` | "Ss_25_10_30" → "Concrete Wall Systems" |
| `IfcClassificationReference.Name` | Human name | `Item.item_name` | "Concrete wall systems" |
| `element.is_a()` | IFC entity type | Fallback group | "IfcWall" → "Walls" |
| `IfcElementType.Name` | Type name | `Item.item_code` | "Basic Wall:200mm Concrete" |

### Data Loss at the Boundary

**IFC data with NO ERPNext equivalent:**
- Geometric representations (IfcShapeRepresentation) — ERPNext has no geometry model
- Spatial relationships (IfcRelContainedInSpatialStructure) — ERPNext uses flat item lists
- Material layer composition (IfcMaterialLayerSetUsage) — ERPNext tracks single-material items
- Connection geometry (IfcRelConnectsPathElements) — no structural analysis in ERPNext
- Parametric definitions — ERPNext stores quantities, not parametric rules

**ERPNext data with NO IFC equivalent:**
- Pricing and valuation rates — IFC has no cost schema
- Warehouse and bin locations — IFC tracks building locations, not stock locations
- Supplier information — IFC has `IfcOrganization` but not procurement workflows
- Serial numbers and batch tracking — no equivalent in IFC
- Workflow states (Draft, Submitted, Cancelled) — no equivalent in IFC

### Frappe API: Creating and Submitting a BOM

```python
def create_bom(building_item: str, bom_items: list[dict]) -> str:
    """Create and submit a BOM in ERPNext. Returns BOM name."""
    bom_data = {
        "doctype": "BOM",
        "item": building_item,
        "items": bom_items,  # [{"item_code": ..., "qty": ..., "uom": ...}, ...]
        "rm_cost_as_per": "Valuation Rate"
    }
    response = requests.post(
        f"{ERP_URL}/api/resource/BOM",
        json=bom_data,
        headers=HEADERS
    )
    response.raise_for_status()
    bom_name = response.json()["data"]["name"]

    # Submit the BOM (makes it active and immutable)
    requests.put(
        f"{ERP_URL}/api/resource/BOM/{bom_name}",
        json={"docstatus": 1},
        headers=HEADERS
    ).raise_for_status()

    return bom_name
```

---

## Reference Links

- [references/methods.md](references/methods.md) — IfcOpenShell quantity API + ERPNext/Frappe REST API
- [references/examples.md](references/examples.md) — End-to-end IFC quantity to ERPNext BOM pipeline
- [references/anti-patterns.md](references/anti-patterns.md) — Costing integration mistakes

### Official Sources

- https://docs.ifcopenshell.org/ — IfcOpenShell 0.8.x API documentation
- https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/ — IFC4 quantity set definitions
- https://frappeframework.com/docs/user/en/api/rest — Frappe REST API documentation
- https://docs.erpnext.com/docs/user/manual/en/manufacturing/bill-of-materials — ERPNext BOM documentation
- https://docs.erpnext.com/docs/user/manual/en/stock/item — ERPNext Item documentation
