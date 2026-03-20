# crosstech-impl-ifc-erpnext-costing: Anti-Patterns

These are confirmed error patterns when bridging IFC quantity data to ERPNext costing.
Each entry documents the WRONG pattern, the CORRECT pattern, and the WHY.

Sources:
- vooronderzoek-aec-automation.md (sections 1.1–1.5)
- https://docs.ifcopenshell.org/
- https://frappeframework.com/docs/user/en/api/rest
- https://docs.erpnext.com/docs/user/manual/en/manufacturing/bill-of-materials

---

## AP-001: Ignoring Unit Scale When Extracting Quantities

**WHY this is wrong**: IFC files can use millimeters, centimeters, inches, or feet as the length unit. If you read `Qto_WallBaseQuantities.Length = 5000` from a millimeter-based file and pass it directly to ERPNext as meters, your BOM shows a 5000-meter wall instead of 5 meters. Area and volume errors compound exponentially (factor 10^6 and 10^9 respectively).

```python
# WRONG — assumes file uses meters
qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
volume = qtos["Qto_WallBaseQuantities"]["GrossVolume"]
bom_item = {"item_code": "WALL-200MM", "qty": volume, "uom": "Cubic Meter"}
# If file uses mm, volume is 1,000,000,000x too large
```

```python
# CORRECT — ALWAYS apply unit scale
unit_scale = ifcopenshell.util.unit.calculate_unit_scale(model)
qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
volume = qtos["Qto_WallBaseQuantities"]["GrossVolume"] * unit_scale**3
bom_item = {"item_code": "WALL-200MM", "qty": volume, "uom": "Cubic Meter"}
```

ALWAYS call `calculate_unit_scale()` once per model and apply it to ALL length, area, and volume values. NEVER assume the IFC file uses meters.

---

## AP-002: Creating One ERPNext Item Per IFC Element Instance

**WHY this is wrong**: A building can have 200 walls of the same type. Creating 200 separate Items in ERPNext floods the item master with duplicates. ERPNext is designed for type-level inventory — one Item represents a material/product type, not individual instances.

```python
# WRONG — one Item per wall instance
for wall in model.by_type("IfcWall"):
    item_code = f"WALL-{wall.GlobalId}"  # Unique per instance
    create_item(item_code, wall.Name, "Walls", "Cubic Meter")
# Result: 200 nearly identical Items cluttering ERPNext
```

```python
# CORRECT — one Item per wall TYPE, aggregate quantities
from collections import defaultdict
type_volumes = defaultdict(float)

for wall in model.by_type("IfcWall"):
    wall_type = ifcopenshell.util.element.get_type(wall)
    type_name = wall_type.Name if wall_type else "UNTYPED_WALL"
    qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
    volume = qtos.get("Qto_WallBaseQuantities", {}).get("GrossVolume", 0)
    type_volumes[type_name] += volume * unit_scale**3

for type_name, total_volume in type_volumes.items():
    create_item(sanitize(type_name), type_name, "Walls", "Cubic Meter")
# Result: ~5 Items representing wall types, with aggregated quantities
```

ALWAYS group IFC elements by their `IfcElementType` before creating ERPNext Items. NEVER create Items at the instance level.

---

## AP-003: Using GrossVolume for Composite/Layered Elements

**WHY this is wrong**: A composite wall (e.g., brick + insulation + plaster) has a `GrossVolume` that includes ALL layers. If you use this volume to cost concrete, you are costing the insulation and plaster as concrete too. The cost estimate will be inflated by 20-50% depending on the wall composition.

```python
# WRONG — uses gross volume for material costing
volume = qtos["Qto_WallBaseQuantities"]["GrossVolume"]
bom_items.append({"item_code": "CONCRETE", "qty": volume, "uom": "Cubic Meter"})
# Includes insulation, plaster, and air gaps as "concrete"
```

```python
# CORRECT — use material-aware quantities
# Option 1: Use NetVolume (excludes voids but still includes all layers)
net_volume = qtos["Qto_WallBaseQuantities"].get("NetVolume", 0)

# Option 2: Extract per-material quantities from IfcMaterialLayerSetUsage
material_usage = ifcopenshell.util.element.get_material(wall)
if material_usage and material_usage.is_a("IfcMaterialLayerSetUsage"):
    layer_set = material_usage.ForLayerSet
    for layer in layer_set.MaterialLayers:
        material_name = layer.Material.Name  # "Concrete", "Insulation", etc.
        layer_thickness = layer.LayerThickness * unit_scale
        layer_volume = qtos["Qto_WallBaseQuantities"].get("GrossSideArea", 0) * unit_scale**2 * layer_thickness
        bom_items.append({
            "item_code": sanitize(material_name),
            "qty": round(layer_volume, 4),
            "uom": "Cubic Meter"
        })
```

For composite elements, ALWAYS extract per-material quantities from `IfcMaterialLayerSetUsage`. NEVER use `GrossVolume` as a proxy for a single material's volume.

---

## AP-004: Submitting BOM Before Verifying All Items Exist

**WHY this is wrong**: ERPNext validates all linked Items when a BOM is submitted (docstatus=1). If any `item_code` in the BOM's child table does not exist as an Item, the submission fails with a `LinkValidationError`. The BOM is left in Draft state, and there is no partial submission — ALL items must exist.

```python
# WRONG — creates BOM and submits without checking Items
bom_data = {
    "doctype": "BOM",
    "item": "BUILDING-01",
    "items": [
        {"item_code": "CONC-WALL-200MM", "qty": 45.6, "uom": "Cubic Meter"},
        {"item_code": "REBAR-B500B", "qty": 3200, "uom": "Kg"},  # Does this Item exist?
    ]
}
response = requests.post(f"{ERP_URL}/api/resource/BOM", json=bom_data, headers=HEADERS)
# May fail at creation or at submission if Items are missing
```

```python
# CORRECT — verify ALL Items exist before creating BOM
required_items = ["CONC-WALL-200MM", "REBAR-B500B", "BUILDING-01"]
for item_code in required_items:
    check = requests.get(f"{ERP_URL}/api/resource/Item/{item_code}", headers=HEADERS)
    if check.status_code != 200:
        raise ValueError(f"Item '{item_code}' does not exist in ERPNext. Create it first.")

# Now safe to create and submit BOM
```

ALWAYS verify that every `item_code` referenced in a BOM exists as an Item in ERPNext before creating the BOM. NEVER assume Items were created successfully without checking.

---

## AP-005: Treating the IfcOpenShell "id" Key as a Quantity

**WHY this is wrong**: `get_psets()` includes a special `"id"` key in every property/quantity set dictionary. This key holds the IfcOpenShell entity ID (an integer), NOT a quantity value. If you iterate all key-value pairs without filtering `"id"`, you create phantom quantities in your BOM.

```python
# WRONG — includes "id" as a quantity
qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
for qto_name, quantities in qtos.items():
    for qty_name, qty_value in quantities.items():
        bom_items.append({"qty_name": qty_name, "qty_value": qty_value})
# Includes {"qty_name": "id", "qty_value": 12345} — NOT a real quantity
```

```python
# CORRECT — filter out the "id" key
qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
for qto_name, quantities in qtos.items():
    for qty_name, qty_value in quantities.items():
        if qty_name == "id":
            continue  # ALWAYS skip the entity ID
        if qty_value is None or qty_value == 0:
            continue  # Skip empty values
        bom_items.append({"qty_name": qty_name, "qty_value": qty_value})
```

ALWAYS filter out the `"id"` key when iterating quantity dictionaries from `get_psets()`. NEVER treat it as a quantity value.

---

## AP-006: Using Case-Incorrect UOM Names in ERPNext

**WHY this is wrong**: ERPNext UOM names are case-sensitive Link fields. `Meter` is a valid UOM, but `meter`, `METER`, and `mtr` are NOT. The API returns a `LinkValidationError` that may not clearly indicate the UOM is the problem.

```python
# WRONG — incorrect UOM casing
bom_items.append({"item_code": "WALL-01", "qty": 5.0, "uom": "cubic meter"})
# ERPNext rejects: "cubic meter" is not a valid UOM

# WRONG — abbreviated UOM
bom_items.append({"item_code": "WALL-01", "qty": 5.0, "uom": "m3"})
# ERPNext rejects: "m3" is not a valid UOM name
```

```python
# CORRECT — exact ERPNext UOM names
UOM_MAP = {
    "length": "Meter",
    "area": "Square Meter",
    "volume": "Cubic Meter",
    "weight": "Kg",
    "count": "Nos",
}
bom_items.append({"item_code": "WALL-01", "qty": 5.0, "uom": "Cubic Meter"})
```

ALWAYS use the exact ERPNext UOM name strings: `Meter`, `Square Meter`, `Cubic Meter`, `Kg`, `Nos`. NEVER use abbreviations, symbols, or different casing.

---

## AP-007: Not Handling Missing Quantity Sets

**WHY this is wrong**: Many IFC files exported from BIM tools do NOT include quantity sets (`Qto_` sets). Some tools only export property sets. Some export quantity sets with all-zero values. If your pipeline assumes every element has quantities, it crashes or creates meaningless zero-quantity BOM Items.

```python
# WRONG — assumes quantities exist
for wall in model.by_type("IfcWall"):
    qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
    volume = qtos["Qto_WallBaseQuantities"]["GrossVolume"]  # KeyError if no Qto_ set
```

```python
# CORRECT — handle missing quantities gracefully
for wall in model.by_type("IfcWall"):
    qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
    qto = qtos.get("Qto_WallBaseQuantities")
    if qto is None:
        print(f"WARNING: Wall #{wall.id()} has no Qto_WallBaseQuantities — skipping")
        continue
    volume = qto.get("GrossVolume", 0)
    if volume <= 0:
        print(f"WARNING: Wall #{wall.id()} has zero/negative volume — skipping")
        continue
```

ALWAYS use `.get()` with a default value when accessing quantity sets and individual quantities. NEVER assume quantity sets exist or contain non-zero values.

---

## AP-008: Flooding ERPNext API Without Rate Limiting

**WHY this is wrong**: A large IFC model can have 10,000+ elements resulting in 500+ unique types. Creating 500 Items followed by a BOM in rapid succession exceeds Frappe's default rate limit (300 requests per 5 minutes). The API returns `429 Too Many Requests` and your pipeline fails mid-execution, leaving partially created data.

```python
# WRONG — no rate limiting
for type_name in type_data:
    requests.post(f"{ERP_URL}/api/resource/Item", json=item_data, headers=HEADERS)
# 429 error after ~300 rapid requests
```

```python
# CORRECT — batch creation with rate limiting
import time

items_to_create = [...]  # List of item dicts

# Option 1: Rate-limited sequential creation
for i, item_data in enumerate(items_to_create):
    response = requests.post(f"{ERP_URL}/api/resource/Item", json=item_data, headers=HEADERS)
    if response.status_code == 429:
        retry_after = int(response.headers.get("Retry-After", 60))
        time.sleep(retry_after)
        response = requests.post(f"{ERP_URL}/api/resource/Item", json=item_data, headers=HEADERS)
    response.raise_for_status()

# Option 2: Batch insert (preferred for 50+ items)
response = requests.post(
    f"{ERP_URL}/api/method/frappe.client.insert_many",
    json={"docs": items_to_create},
    headers=HEADERS
)
```

For bulk operations (50+ Items), ALWAYS use `frappe.client.insert_many` for batch creation. NEVER send hundreds of sequential POST requests without rate limiting.
