# methods.md — n8n Node Reference for AEC Pipelines

## n8n Core Node Signatures

### Webhook Node (`n8n-nodes-base.webhook`)

**Purpose:** Entry point for AEC event-driven pipelines. Receives IFC uploads, Speckle commit hooks, and external triggers.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `httpMethod` | string | YES | `GET`, `POST`, `PUT`, `DELETE` |
| `path` | string | YES | URL path segment (e.g., `ifc-upload`) |
| `authentication` | string | NO | `none`, `basicAuth`, `headerAuth` |
| `responseMode` | string | NO | `onReceived` (immediate) or `lastNode` (wait for pipeline) |
| `binaryPropertyName` | string | NO | Property name for uploaded files (default: `data`) |
| `options.rawBody` | boolean | NO | Keep raw request body (needed for binary uploads) |

**Output:** JSON object with `headers`, `params`, `query`, `body` fields. Binary data in `$binary[propertyName]`.

**AEC usage:** Set `httpMethod: POST`, `path: ifc-upload`, `binaryPropertyName: ifc_file`, `options.rawBody: true` for IFC file uploads.

---

### HTTP Request Node (`n8n-nodes-base.httpRequest`)

**Purpose:** Call external AEC service REST APIs (ERPNext, Speckle, IfcPipeline).

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `method` | string | YES | `GET`, `POST`, `PUT`, `PATCH`, `DELETE` |
| `url` | string | YES | Full URL to endpoint |
| `authentication` | string | NO | `none`, `genericCredentialType`, `predefinedCredentialType` |
| `sendHeaders` | boolean | NO | Send custom headers |
| `sendBody` | boolean | NO | Send request body |
| `specifyBody` | string | NO | `keypair` or `json` |
| `jsonBody` | string | NO | JSON string body (supports expressions) |
| `timeout` | number | NO | Request timeout in ms |

**ERPNext API patterns:**

```
# Create Item
POST https://erp.example.com/api/resource/Item
Body: { "data": { "item_code": "...", "item_group": "...", "uom": "..." } }

# Create BOM
POST https://erp.example.com/api/resource/BOM
Body: { "data": { "item": "...", "items": [...], "quantity": 1 } }

# Submit BOM (change docstatus to 1)
PUT https://erp.example.com/api/resource/BOM/{name}
Body: { "data": { "docstatus": 1 } }

# List Items with filters
GET https://erp.example.com/api/resource/Item?filters=[["item_group","=","Concrete"]]&fields=["item_code","item_name"]
```

**Speckle GraphQL API:**

```
POST https://speckle.example.com/graphql
Headers: Authorization: Bearer {PAT}
Body: { "query": "{ stream(id: \"...\") { name commits { items { id message } } } }" }
```

---

### Execute Command Node (`n8n-nodes-base.executeCommand`)

**Purpose:** Run IfcOpenShell Python scripts, CLI tools, and system commands on the n8n host.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `command` | string | YES | Shell command to execute |
| `timeout` | number | NO | Execution timeout in ms (default: 10000) |

**Output:** JSON with `stdout`, `stderr`, `exitCode` fields.

**AEC usage patterns:**

```bash
# Validate IFC file
python3 /scripts/validate_ifc.py --input "/data/models/{{ $json.file_name }}"

# Extract quantities to JSON
python3 /scripts/extract_quantities.py --input "/data/models/{{ $json.file_name }}" --output /tmp/quantities.json

# Convert IFC to GLB (using IfcConvert CLI)
IfcConvert "/data/models/{{ $json.file_name }}" "/data/output/{{ $json.file_name }}.glb"

# Run IDS validation
python3 -m ifctester "/data/models/{{ $json.file_name }}" "/specs/{{ $json.ids_file }}"
```

**ALWAYS** set timeout to at least 120000 for IFC operations. Default 10000ms is insufficient.

---

### Code Node (`n8n-nodes-base.code`)

**Purpose:** Transform data between AEC service formats using JavaScript.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `jsCode` | string | YES | JavaScript code to execute |
| `mode` | string | NO | `runOnceForAllItems` or `runOnceForEachItem` |

**Available variables:**
- `$input.all()` — all input items
- `$input.first()` — first input item
- `$json` — current item's JSON data (in `runOnceForEachItem` mode)
- `$binary` — current item's binary data

**AEC transformation example (IFC quantities → ERPNext BOM items):**

```javascript
// Transform IfcOpenShell quantity output to ERPNext BOM format
const quantities = $input.first().json;
const bomItems = quantities.elements.map(elem => ({
  item_code: elem.type + '_' + elem.material,
  qty: elem.quantities.NetVolume || elem.quantities.NetArea || 1,
  uom: elem.quantities.NetVolume ? 'm3' : (elem.quantities.NetArea ? 'm2' : 'Nos'),
  rate: 0  // ERPNext fills from Valuation Rate
}));

return [{ json: { item_code: quantities.project_code, bom_items: bomItems } }];
```

---

### Schedule Trigger Node (`n8n-nodes-base.scheduleTrigger`)

**Purpose:** Start AEC pipelines on a schedule (nightly checks, periodic exports).

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `rule.interval` | array | YES | Cron rules or interval definitions |

**Common AEC schedules:**

| Schedule | Cron | Use Case |
|----------|------|----------|
| Nightly at 02:00 | `0 2 * * *` | BIM quality checks |
| Every Monday 08:00 | `0 8 * * 1` | Weekly model report |
| Every 6 hours | `0 */6 * * *` | Speckle sync polling |
| First of month | `0 9 1 * *` | Monthly cost reconciliation |

---

### If Node (`n8n-nodes-base.if`)

**Purpose:** Route pipeline based on validation results, file sizes, or service responses.

**AEC routing conditions:**

| Condition | Expression | Route |
|-----------|-----------|-------|
| IFC valid | `{{ $json.exitCode }} === 0` | Continue processing |
| File too large | `{{ $json.file_size }} > 500000000` | Route to async processor |
| Schema version | `{{ $json.schema }} === "IFC4"` | Route to IFC4 handler |
| BOM exists | `{{ $json.data.length }} > 0` | Update existing BOM |

---

### Switch Node (`n8n-nodes-base.switch`)

**Purpose:** Multi-way routing based on IFC schema version, element type, or processing stage.

**AEC routing example:**

| Value | Route | Action |
|-------|-------|--------|
| `IFC2X3` | Output 0 | Legacy conversion pipeline |
| `IFC4` | Output 1 | Standard processing |
| `IFC4X3` | Output 2 | Infrastructure-specific handling |

---

### Execute Sub-workflow Node (`n8n-nodes-base.executeWorkflow`)

**Purpose:** Call reusable pipeline stages (validation, notification, BOM creation).

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `workflowId` | string | YES | ID or name of sub-workflow |
| `mode` | string | NO | `once` or `each` (per item) |

**Recommended AEC sub-workflows:**

1. `ifc-validation` — validate IFC file, return pass/fail + details
2. `erpnext-bom-create` — create/update ERPNext BOM from quantity data
3. `notify-team` — send email + Slack notification with formatted message
4. `speckle-fetch-commit` — retrieve Speckle commit details via GraphQL

---

## AEC Service API Endpoints Reference

### ERPNext REST API

| Operation | Method | Endpoint | Body |
|-----------|--------|----------|------|
| Create Item | POST | `/api/resource/Item` | `{ "data": { "item_code": "...", ... } }` |
| Create BOM | POST | `/api/resource/BOM` | `{ "data": { "item": "...", "items": [...] } }` |
| Submit BOM | PUT | `/api/resource/BOM/{name}` | `{ "data": { "docstatus": 1 } }` |
| Get Item | GET | `/api/resource/Item/{name}` | — |
| List with filter | GET | `/api/resource/Item?filters=[...]&fields=[...]` | — |

**Authentication header:** `Authorization: token {api_key}:{api_secret}`

### Speckle GraphQL API

| Operation | Query |
|-----------|-------|
| List streams | `{ streams { items { id name } } }` |
| Get commit | `{ stream(id: "...") { commit(id: "...") { message referencedObject } } }` |
| Get object | `{ stream(id: "...") { object(id: "...") { data totalChildrenCount } } }` |

**Authentication header:** `Authorization: Bearer {personal_access_token}`

### IfcPipeline REST API

| Operation | Method | Endpoint | Input |
|-----------|--------|----------|-------|
| Convert IFC | POST | `/ifcconvert` | `{ "ifc_file": "path", "format": "glb" }` |
| Validate (IDS) | POST | `/ifctester` | `{ "ifc_file": "path", "ids_file": "path" }` |
| Clash detection | POST | `/ifcclash` | `{ "ifc_file_a": "path", "ifc_file_b": "path" }` |
| Quantity takeoff | POST | `/calculate-qtos` | `{ "ifc_file": "path" }` |
| Model diff | POST | `/ifcdiff` | `{ "old_file": "path", "new_file": "path" }` |
| Export JSON | POST | `/ifc2json` | `{ "ifc_file": "path" }` |
| CSV export | POST | `/ifccsv` | `{ "ifc_file": "path" }` |

**Authentication header:** `X-API-Key: {api_key}`
