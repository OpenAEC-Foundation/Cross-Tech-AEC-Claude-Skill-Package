# anti-patterns.md — n8n AEC Pipeline Mistakes

## Anti-Pattern 1: Processing IFC Geometry Inside Code Nodes

**What happens:** Developer writes IfcOpenShell-style geometry parsing logic inside an n8n Code node.

**Why it fails:**
- n8n Code nodes run JavaScript, not Python — IfcOpenShell is a Python library
- Code nodes have a 10-second default execution timeout
- Code nodes have limited memory (typically 256MB per execution)
- IFC geometry processing requires OpenCASCADE, which is unavailable in the n8n runtime

**Wrong:**
```javascript
// Inside n8n Code node — THIS DOES NOT WORK
const ifcopenshell = require('ifcopenshell');  // Does not exist in JS
const model = ifcopenshell.open('/data/model.ifc');
```

**Correct:**
```json
{
  "name": "Process IFC",
  "type": "n8n-nodes-base.executeCommand",
  "parameters": {
    "command": "python3 /scripts/extract_quantities.py --input /data/model.ifc --format json",
    "timeout": 300000
  }
}
```

**Rule:** ALWAYS delegate IFC processing to Execute Command (Python scripts) or HTTP Request (IfcPipeline service). NEVER attempt IFC operations in Code nodes.

---

## Anti-Pattern 2: Embedding Binary IFC Data in JSON Flow

**What happens:** Developer reads an IFC file and passes its content as a JSON string field between nodes.

**Why it fails:**
- IFC files range from 1MB to 2GB — embedding in JSON causes memory exhaustion
- Base64 encoding increases size by 33%
- n8n's execution data storage has size limits (configurable, but defaults to ~16MB)
- Workflow execution history becomes bloated and slow to load

**Wrong:**
```javascript
// Inside Code node
const fs = require('fs');
const content = fs.readFileSync('/data/model.ifc', 'base64');
return [{ json: { ifc_content: content } }];  // 500MB string in JSON!
```

**Correct:**
```javascript
// Pass file PATH, not file CONTENT
return [{ json: { file_path: '/data/models/project-v2.ifc' } }];
```

**Rule:** ALWAYS pass file paths between nodes. NEVER read binary file content into JSON data flow. Use n8n's `$binary` properties for binary file transfers between webhook/HTTP nodes.

---

## Anti-Pattern 3: No Error Handling in Production Pipelines

**What happens:** Developer deploys AEC pipeline without Error Trigger workflow or per-node error handling.

**Why it fails:**
- IFC files from external parties frequently contain schema violations
- AEC service APIs (ERPNext, Speckle) have rate limits and downtime
- Network timeouts between containers are common in Docker stacks
- A single failed node stops the entire pipeline silently

**Wrong:**
```json
{
  "nodes": [
    { "name": "Webhook", "type": "n8n-nodes-base.webhook" },
    { "name": "Process", "type": "n8n-nodes-base.executeCommand" },
    { "name": "Upload", "type": "n8n-nodes-base.httpRequest" }
  ]
}
```

**Correct:**
- Add `continueOnFail: true` on non-critical nodes (notifications, logging)
- Create a separate Error Trigger workflow that catches all unhandled failures
- Add If nodes after each Execute Command to check `exitCode`
- Set explicit timeouts on HTTP Request and Execute Command nodes

**Rule:** ALWAYS create an Error Trigger workflow for production AEC pipelines. ALWAYS check `exitCode` after Execute Command nodes. ALWAYS set explicit timeouts.

---

## Anti-Pattern 4: Hardcoded Credentials in Workflow JSON

**What happens:** Developer puts API keys, tokens, and passwords directly in node parameters.

**Why it fails:**
- n8n workflows are exported as JSON for backup and sharing
- Exported JSON includes ALL parameter values — including hardcoded secrets
- Team members import shared workflows and accidentally inherit production credentials
- No credential rotation without editing every workflow that uses the credential

**Wrong:**
```json
{
  "name": "Call ERPNext",
  "type": "n8n-nodes-base.httpRequest",
  "parameters": {
    "url": "https://erp.example.com/api/resource/Item",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "Authorization", "value": "token abc123:secret456" }
      ]
    }
  }
}
```

**Correct:**
```json
{
  "name": "Call ERPNext",
  "type": "n8n-nodes-base.httpRequest",
  "parameters": {
    "url": "https://erp.example.com/api/resource/Item",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpHeaderAuth"
  },
  "credentials": {
    "httpHeaderAuth": { "id": "erpnext-token", "name": "ERPNext API Token" }
  }
}
```

**Rule:** ALWAYS use n8n's credential store. NEVER put secrets in node parameters. Configure credentials via n8n UI (Settings > Credentials) before workflow deployment.

---

## Anti-Pattern 5: Default Timeouts for IFC Operations

**What happens:** Developer leaves Execute Command or HTTP Request nodes at default timeout when calling IFC processing services.

**Why it fails:**
- Execute Command default timeout: 10,000ms (10 seconds)
- HTTP Request default timeout: 300,000ms (5 minutes) but effectively limited by n8n execution timeout
- IFC validation on a 200MB model: 30-120 seconds
- IFC quantity extraction on a 500MB model: 2-10 minutes
- IFC clash detection between two models: 5-30 minutes
- Silent timeout = silent failure = lost pipeline execution

**Wrong:**
```json
{
  "name": "Validate IFC",
  "type": "n8n-nodes-base.executeCommand",
  "parameters": {
    "command": "python3 /scripts/validate_ifc.py --input /data/large-model.ifc"
  }
}
```

**Correct:**
```json
{
  "name": "Validate IFC",
  "type": "n8n-nodes-base.executeCommand",
  "parameters": {
    "command": "python3 /scripts/validate_ifc.py --input /data/large-model.ifc",
    "timeout": 300000
  }
}
```

**Timeout guidelines for AEC operations:**

| Operation | Recommended Timeout | Why |
|-----------|-------------------|-----|
| IFC validation | 120,000ms (2 min) | Schema checks are fast, but file parsing takes time |
| Quantity extraction | 300,000ms (5 min) | Iterates all elements, reads property sets |
| Clash detection | 1,800,000ms (30 min) | Geometric computation on two full models |
| IFC conversion (to GLB) | 600,000ms (10 min) | Geometry tessellation is CPU-intensive |
| ERPNext API call | 30,000ms (30 sec) | Network + server processing |
| Speckle GraphQL | 30,000ms (30 sec) | Network + query complexity |

**Rule:** ALWAYS set explicit timeouts on Execute Command and HTTP Request nodes based on the operation type. NEVER rely on default timeouts for IFC processing.

---

## Anti-Pattern 6: Monolithic Pipelines Instead of Sub-workflows

**What happens:** Developer builds a single 20+ node workflow for the entire AEC pipeline.

**Why it fails:**
- Debugging becomes difficult — one failed node in a chain of 20 is hard to identify
- No reusability — validation logic duplicated across pipelines
- Execution history shows one massive execution instead of modular stages
- Cannot retry individual stages — must rerun the entire pipeline
- Workflow JSON becomes unreadable and hard to version-control

**Wrong:** One workflow with 25 nodes: Webhook → Save → Validate → Extract → Transform → Create Items → Create BOM → Submit → Email → Log → ...

**Correct:** Three sub-workflows called from a parent:
1. `ifc-validation` sub-workflow (5 nodes)
2. `erpnext-bom-create` sub-workflow (8 nodes)
3. `notify-team` sub-workflow (3 nodes)

Parent workflow:
```
[Webhook] → [Execute Sub-workflow: ifc-validation]
    → [If: valid?]
        ├── YES → [Execute Sub-workflow: erpnext-bom-create]
        │         → [Execute Sub-workflow: notify-team]
        └── NO  → [Execute Sub-workflow: notify-team (failure)]
```

**Rule:** ALWAYS break pipelines with more than 10 nodes into sub-workflows. ALWAYS extract reusable logic (validation, notification, API calls) into dedicated sub-workflows.

---

## Anti-Pattern 7: Polling Speckle Instead of Using Webhooks

**What happens:** Developer creates a Schedule Trigger that polls the Speckle API every 5 minutes to check for new commits.

**Why it fails:**
- Unnecessary API calls (286/day for 5-minute intervals)
- 5-minute delay between commit and reaction
- Speckle API rate limits can block other legitimate requests
- Wastes n8n execution resources on empty checks

**Wrong:**
```
[Schedule Trigger: every 5 min] → [HTTP Request: Speckle GraphQL] → [Code: compare timestamps] → ...
```

**Correct:**
```
[Webhook: speckle-commit] → [HTTP Request: fetch commit details] → ...
```

Configure the Speckle webhook in Speckle Server settings:
- URL: `https://n8n.example.com/webhook/speckle-commit`
- Events: `commit_create`
- Stream: select specific stream or all streams

**Rule:** ALWAYS use Speckle webhooks for event-driven pipelines. NEVER poll Speckle for changes. Polling is acceptable ONLY when webhook delivery is unreliable and as a fallback mechanism.

---

## Anti-Pattern 8: Ignoring n8n Execution Data Size Limits

**What happens:** Developer builds a pipeline that extracts ALL properties from ALL elements in a large IFC model and passes them through n8n's data flow.

**Why it fails:**
- A 100MB IFC model can have 50,000+ elements with 20+ properties each
- Extracted JSON can be 500MB+ when all properties are included
- n8n stores execution data in its database — large executions slow down the entire instance
- Memory consumption during execution causes OOM kills in Docker

**Wrong:**
```bash
# Extract ALL data from entire model
python3 /scripts/extract_all.py --input model.ifc --format json
# Returns 500MB JSON with every property of every element
```

**Correct:**
```bash
# Extract only needed data, filtered by element type
python3 /scripts/extract_quantities.py --input model.ifc --types "IfcWall,IfcSlab,IfcBeam" --properties "Qto_*" --format json
# Returns 2MB JSON with quantities for structural elements only
```

**Rule:** ALWAYS filter IFC data extraction to only the element types and properties needed by the downstream service. NEVER extract the complete IFC property tree into n8n's data flow.
