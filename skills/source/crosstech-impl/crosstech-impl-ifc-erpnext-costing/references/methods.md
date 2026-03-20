# crosstech-impl-ifc-erpnext-costing: Methods Reference

Sources:
- https://docs.ifcopenshell.org/
- https://standards.buildingsmart.org/IFC/RELEASE/IFC4/ADD2_TC1/HTML/
- https://frappeframework.com/docs/user/en/api/rest
- https://docs.erpnext.com/docs/user/manual/en/manufacturing/bill-of-materials

---

## 1. IfcOpenShell Quantity Extraction API

### 1.1 ifcopenshell.util.element.get_psets()

The primary function for extracting both property sets and quantity sets from IFC elements.

**Signature:**
```python
ifcopenshell.util.element.get_psets(
    element,           # IfcProduct or IfcTypeProduct instance
    psets_only=False,  # If True, return only IfcPropertySet (exclude quantities)
    qtos_only=False,   # If True, return only IfcElementQuantity (exclude properties)
    should_inherit=True # If True, include type-level properties
) -> dict[str, dict[str, Any]]
```

**Return format:**
```python
{
    "Qto_WallBaseQuantities": {
        "id": 12345,       # IfcOpenShell entity ID (ALWAYS present)
        "Length": 5.0,
        "Height": 3.0,
        "Width": 0.2,
        "GrossSideArea": 15.0,
        "NetSideArea": 13.5,
        "GrossVolume": 3.0,
        "NetVolume": 2.7,
        "GrossWeight": 7200.0
    }
}
```

**CRITICAL**: The `"id"` key is ALWAYS present in each sub-dictionary. NEVER treat it as a quantity value. ALWAYS filter it out when iterating quantity values.

### 1.2 ifcopenshell.util.element.get_type()

Returns the IfcElementType associated with an element instance.

**Signature:**
```python
ifcopenshell.util.element.get_type(element) -> Optional[entity_instance]
```

**Usage:**
```python
wall_type = ifcopenshell.util.element.get_type(wall)
if wall_type:
    type_name = wall_type.Name  # e.g., "Basic Wall:200mm Concrete"
```

Returns `None` if the element has no type assignment.

### 1.3 ifcopenshell.util.unit.calculate_unit_scale()

Returns a float that converts file length units to meters.

**Signature:**
```python
ifcopenshell.util.unit.calculate_unit_scale(ifc_file) -> float
```

**Return values:**
| File Unit | Return Value | Meaning |
|-----------|-------------|---------|
| Meters | 1.0 | No conversion needed |
| Millimeters | 0.001 | Multiply lengths by 0.001 |
| Centimeters | 0.01 | Multiply lengths by 0.01 |
| Inches | 0.0254 | Multiply lengths by 0.0254 |
| Feet | 0.3048 | Multiply lengths by 0.3048 |

**Unit scale for derived quantities:**
- Length values: multiply by `unit_scale`
- Area values: multiply by `unit_scale ** 2`
- Volume values: multiply by `unit_scale ** 3`
- Count values: NO conversion needed
- Weight values: separate unit check required (see 1.4)

### 1.4 ifcopenshell.util.classification.get_references()

Returns classification references attached to an element.

**Signature:**
```python
ifcopenshell.util.classification.get_references(element) -> list[entity_instance]
```

**Usage:**
```python
refs = ifcopenshell.util.classification.get_references(wall)
for ref in refs:
    system = ref.ReferencedSource.Name  # "Uniclass", "OmniClass", etc.
    code = ref.Identification            # "Ss_25_10_30"
    name = ref.Name                      # "Concrete wall systems"
```

### 1.5 IFC Quantity Types — Complete Reference

Each quantity type stores its value in a specific attribute:

| IFC Type | Value Attribute | Python Access | Unit Dimension |
|----------|----------------|---------------|----------------|
| `IfcQuantityLength` | `LengthValue` | `float` | Length (m) |
| `IfcQuantityArea` | `AreaValue` | `float` | Area (m2) |
| `IfcQuantityVolume` | `VolumeValue` | `float` | Volume (m3) |
| `IfcQuantityCount` | `CountValue` | `int` | Dimensionless |
| `IfcQuantityWeight` | `WeightValue` | `float` | Mass (kg) |

**NOTE**: `IfcQuantityTime` also exists (for scheduling) but is NOT relevant to costing.

When using `get_psets(qtos_only=True)`, the return dict flattens these — each quantity's `Name` becomes the key and the numeric value becomes the value. The underlying type information is lost in this flattening. If you need to know whether a value is a length vs. area, you MUST inspect the raw IFC entity:

```python
for qto_entity in model.by_type("IfcElementQuantity"):
    for quantity in qto_entity.Quantities:
        print(quantity.is_a())  # "IfcQuantityLength", "IfcQuantityArea", etc.
        print(quantity.Name)    # "Length", "GrossSideArea", etc.
```

---

## 2. ERPNext / Frappe REST API

### 2.1 Authentication

ERPNext uses token-based authentication for API access.

**Header format:**
```
Authorization: token {api_key}:{api_secret}
Content-Type: application/json
```

**Generate tokens**: ERPNext Settings → API Access → Generate Keys. ALWAYS use a dedicated API user with restricted permissions (Item, BOM, Project only).

### 2.2 CRUD Operations

**Create (POST):**
```
POST /api/resource/{DocType}
Body: {"doctype": "{DocType}", "field1": "value1", ...}
Returns: {"data": {"name": "...", ...}}
```

**Read (GET):**
```
GET /api/resource/{DocType}/{name}
Returns: {"data": {"name": "...", ...}}
```

**Update (PUT):**
```
PUT /api/resource/{DocType}/{name}
Body: {"field1": "new_value"}
Returns: {"data": {"name": "...", ...}}
```

**List (GET):**
```
GET /api/resource/{DocType}?filters=[["field","=","value"]]&fields=["name","field1"]&limit_page_length=100
```

**Submit (workflow transition):**
```
PUT /api/resource/{DocType}/{name}
Body: {"docstatus": 1}
```

### 2.3 Item DocType — Key Fields

| Field | Type | Required | Mapping Source |
|-------|------|:--------:|---------------|
| `item_code` | String (140) | YES | IfcElementType.Name (sanitized) |
| `item_name` | String (140) | YES | IfcElementType.Name |
| `item_group` | Link (Item Group) | YES | IfcClassificationReference or element type fallback |
| `stock_uom` | Link (UOM) | YES | IFC unit → ERPNext UOM name |
| `description` | Text | NO | IFC element Name + Description |
| `is_stock_item` | Check | NO | Default: 1 for materials, 0 for services |
| `has_batch_no` | Check | NO | Default: 0 (IFC has no batch tracking) |

### 2.4 BOM DocType — Key Fields

| Field | Type | Required | Notes |
|-------|------|:--------:|-------|
| `item` | Link (Item) | YES | The parent item this BOM describes |
| `items` | Table (BOM Item) | YES | Child rows with components |
| `quantity` | Float | YES | Default: 1 (quantity of parent item this BOM produces) |
| `rm_cost_as_per` | Select | NO | "Valuation Rate", "Last Purchase Rate", "Price List" |
| `is_default` | Check | NO | Whether this is the default BOM for the item |
| `is_active` | Check | NO | Default: 1 |

### 2.5 BOM Item (Child Table) — Key Fields

| Field | Type | Required | Mapping Source |
|-------|------|:--------:|---------------|
| `item_code` | Link (Item) | YES | The component material item |
| `qty` | Float | YES | Aggregated IFC quantity (after unit conversion) |
| `uom` | Link (UOM) | YES | ERPNext UOM matching the quantity type |
| `rate` | Currency | NO | Unit cost (populated from ERPNext pricing) |
| `amount` | Currency | NO | Auto-calculated: qty * rate |

### 2.6 ERPNext UOM Names (Exact Strings)

ERPNext ships with predefined UOM names. These are the EXACT strings to use:

| UOM Name | Symbol | Maps From IFC |
|----------|--------|--------------|
| `Meter` | m | IfcQuantityLength |
| `Square Meter` | m2 | IfcQuantityArea |
| `Cubic Meter` | m3 | IfcQuantityVolume |
| `Kg` | kg | IfcQuantityWeight |
| `Nos` | nos | IfcQuantityCount |
| `Unit` | unit | Generic fallback |

**CRITICAL**: UOM names are case-sensitive in ERPNext. `Meter` works, `meter` does NOT. `Kg` works, `KG` does NOT.

### 2.7 Rate Limiting

The Frappe framework enforces rate limits. Default: 300 requests per 5 minutes for API users.

- ALWAYS implement retry logic with exponential backoff
- For bulk operations (50+ items), use batch creation via the `/api/method/frappe.client.insert_many` endpoint
- Monitor `X-RateLimit-Remaining` response header

---

## 3. IFC to ERPNext Field Mapping — Complete Table

| IFC Source | IFC Field/Method | ERPNext DocType | ERPNext Field | Notes |
|-----------|-----------------|----------------|---------------|-------|
| `IfcProject` | `.Name` | Project | `project_name` | Top-level mapping |
| `IfcBuildingStorey` | `.Name` | Cost Center | `cost_center_name` | Storey-level cost tracking |
| `IfcElementType` | `.Name` | Item | `item_code` | Sanitize: remove special chars |
| `IfcElementType` | `.Name` | Item | `item_name` | Human-readable name |
| `IfcClassificationReference` | `.Identification` | Item Group | `name` | Create hierarchy if needed |
| `IfcClassificationReference` | `.Name` | Item Group | `name` (fallback) | When code is not human-readable |
| `element.is_a()` | Entity type | Item Group | `name` (fallback) | "IfcWall" → "Walls" |
| `IfcElementQuantity` | quantity value | BOM Item | `qty` | After unit conversion |
| `IfcUnitAssignment` | unit type | BOM Item | `uom` | IFC unit → ERPNext UOM |
| `IfcMaterial` | `.Name` | Item | `item_code` (material) | For material-level costing |
