---
name: crosstech-impl-n8n-aec-pipeline
description: >
  Use when automating AEC workflows with n8n: IFC processing pipelines,
  Speckle webhook integration, model validation, or connecting AEC tools via APIs.
  Prevents the common mistake of running heavy IFC processing inside n8n nodes
  instead of delegating to external services.
  Covers n8n v1.x nodes for AEC, webhook triggers, Execute Command for IfcOpenShell,
  Speckle/ERPNext API integration, and pipeline patterns.
  Keywords: n8n, workflow automation, AEC pipeline, IFC processing, Speckle webhook,
  ERPNext API, model validation, n8n nodes, Execute Command.
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x with AEC service endpoints."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-impl-n8n-aec-pipeline

## Quick Reference

### Core n8n Nodes for AEC Automation

| Node | Type Identifier | AEC Purpose |
|------|----------------|-------------|
| HTTP Request | `n8n-nodes-base.httpRequest` | Call ERPNext, Speckle, IfcPipeline REST APIs |
| Webhook | `n8n-nodes-base.webhook` | Receive IFC uploads, Speckle commit events |
| Code | `n8n-nodes-base.code` | Transform JSON data, format reports, light parsing |
| Execute Command | `n8n-nodes-base.executeCommand` | Run IfcOpenShell scripts, CLI tools on host |
| Schedule Trigger | `n8n-nodes-base.scheduleTrigger` | Nightly BIM checks, scheduled exports |
| FTP/SFTP | `n8n-nodes-base.ftp` | Upload/download IFC files from project servers |
| If | `n8n-nodes-base.if` | Route on validation pass/fail |
| Switch | `n8n-nodes-base.switch` | Route by IFC schema version or element type |
| Merge | `n8n-nodes-base.merge` | Combine data from multiple AEC sources |
| Execute Sub-workflow | `n8n-nodes-base.executeWorkflow` | Modular reusable pipeline stages |
| Send Email | `n8n-nodes-base.emailSend` | Distribute reports to project teams |

### Critical Warnings

**NEVER** run heavy IFC geometry processing inside n8n Code nodes — n8n runs JavaScript with a 10-second default timeout. ALWAYS delegate to external Python services via Execute Command or HTTP Request.

**NEVER** store IFC files in n8n's JSON data flow — binary files MUST be handled via file system paths or binary properties. n8n nodes pass JSON arrays between them; embedding multi-MB binary data causes memory exhaustion.

**NEVER** hardcode credentials in workflow JSON — ALWAYS use n8n's encrypted credential store. Workflow JSON is exportable and shareable; embedded secrets leak.

**ALWAYS** set explicit timeouts on Execute Command nodes running IfcOpenShell scripts — large IFC files (100MB+) can take minutes to process. The default timeout causes silent failures.

---

## Technology Boundary

### Side A: n8n Workflow Engine (v1.x)

| Aspect | Detail |
|--------|--------|
| Runtime | Node.js 18+ (self-hosted) or n8n Cloud |
| Data model | JSON arrays flowing between nodes |
| Execution | Sequential node execution with branching |
| Triggers | Webhook, Schedule, Manual, External event |
| Extensions | Community nodes via npm (`n8n-nodes-*`) |
| Binary handling | `$binary` object with `fileName`, `mimeType`, `data` |
| Credential storage | AES-256 encrypted in n8n database |
| Docker image | `n8nio/n8n:latest` (or pinned version tag) |

### Side B: AEC Tool APIs (Speckle, ERPNext, IfcOpenShell)

| Service | Protocol | Authentication | Data Format |
|---------|----------|---------------|-------------|
| ERPNext | REST (`/api/resource/:doctype`) | Token: `api_key:api_secret` | JSON |
| Speckle | GraphQL + REST | Bearer token (PAT) | JSON + binary streams |
| IfcOpenShell | CLI / Python scripts | N/A (local execution) | IFC (STEP), JSON output |
| IfcPipeline | REST (FastAPI) | API key header | JSON + binary IFC |
| QGIS Server | WMS/WFS (OGC) | None (internal network) | XML/GML |

### The Bridge: HTTP / Webhook / CLI Integration

n8n connects to AEC services through three integration patterns:

1. **HTTP Request nodes** → REST/GraphQL APIs (Speckle, ERPNext, IfcPipeline)
2. **Webhook nodes** → receive push events from Speckle commits, file uploads
3. **Execute Command nodes** → invoke IfcOpenShell Python scripts on the host

Data transformation happens in Code nodes (JavaScript) between these boundary crossings. The Code node transforms AEC-specific JSON structures into the format expected by the next service.

**Data loss at boundary:** IFC geometry is NEVER transmitted through n8n's JSON pipeline. Only metadata, quantities, and validation results cross the n8n boundary. Geometry stays in IFC files on disk or in dedicated services.

---

## Critical Rules

1. **ALWAYS** delegate IFC processing to external services — n8n is an orchestrator, not a compute engine
2. **ALWAYS** use `$binary` properties for file transfers — NEVER base64-encode IFC files into JSON fields
3. **ALWAYS** configure Error Trigger nodes (`n8n-nodes-base.errorTrigger`) for every production pipeline
4. **ALWAYS** use Execute Sub-workflow for reusable pipeline stages (validation, notification)
5. **NEVER** process IFC geometry in Code nodes — Code nodes run JavaScript with limited memory and timeout
6. **NEVER** expose webhook URLs without authentication in production — use header auth or webhook path secrets
7. **ALWAYS** set `continueOnFail: true` on non-critical nodes to prevent full pipeline abort
8. **ALWAYS** store AEC service credentials in n8n's credential store with the correct type

---

## Decision Tree

```
Need to automate an AEC workflow?
├── Trigger type?
│   ├── File upload event → Webhook node (POST, binary)
│   ├── Speckle model commit → Webhook node (Speckle webhook callback)
│   ├── Scheduled check → Schedule Trigger node (cron expression)
│   └── Manual/API trigger → Manual Trigger or Webhook node
├── Need IFC processing?
│   ├── Light metadata extraction → Execute Command (IfcOpenShell script)
│   ├── Heavy geometry operations → HTTP Request to IfcPipeline service
│   ├── Validation/clash detection → HTTP Request to IfcPipeline `/ifctester` or `/ifcclash`
│   └── Format conversion → HTTP Request to IfcPipeline `/ifcconvert`
├── Need to update external system?
│   ├── ERPNext BOM/Item → HTTP Request (POST /api/resource/:doctype)
│   ├── Speckle stream → HTTP Request (GraphQL mutation)
│   ├── Notification → Send Email or HTTP Request (Slack/Teams webhook)
│   └── File delivery → FTP/SFTP node
└── Error handling?
    ├── Validation failure → If node routes to notification branch
    ├── Service down → Error Trigger → retry or alert
    └── Partial failure → continueOnFail: true on non-critical nodes
```

---

## Essential Patterns

### Pattern 1: IFC Upload → Validation → Extraction → Report

**Flow:** Webhook → Execute Command (validate) → If → Execute Command (extract) → HTTP Request (ERPNext) → Send Email

```
[Webhook: IFC Upload] → [Execute Command: validate_ifc.py]
    → [If: valid?]
        ├── YES → [Execute Command: extract_quantities.py]
        │         → [Code: transform to ERPNext format]
        │         → [HTTP Request: POST /api/resource/BOM]
        │         → [Send Email: success report]
        └── NO  → [Send Email: validation failure report]
```

**Webhook node configuration:**
- Method: POST
- Path: `ifc-upload`
- Binary Property: `ifc_file`
- Options: `rawBody: true`

**Execute Command configuration:**
- Command: `python3 /scripts/validate_ifc.py --input /tmp/{{ $binary.ifc_file.fileName }}`
- ALWAYS use absolute paths to scripts
- ALWAYS pass file paths, not file content

### Pattern 2: Speckle Webhook → Change Detection → Notification

**Flow:** Webhook → HTTP Request (Speckle GraphQL) → Code (diff) → If → HTTP Request (notify)

```
[Webhook: Speckle commit] → [HTTP Request: fetch commit details]
    → [Code: compare with previous commit]
    → [If: significant changes?]
        ├── YES → [HTTP Request: Slack notification]
        └── NO  → [No operation]
```

**Speckle GraphQL query for commit details:**
```graphql
query {
  stream(id: "{{ $json.payload.streamId }}") {
    commit(id: "{{ $json.payload.commitId }}") {
      message
      authorName
      createdAt
      referencedObject
    }
  }
}
```

### Pattern 3: Scheduled BIM Quality Check → Report → Email

**Flow:** Schedule Trigger → Execute Command (quality check) → Code (format) → Send Email

```
[Schedule Trigger: 02:00 daily] → [Execute Command: bim_quality_check.py]
    → [Code: format HTML report]
    → [Send Email: distribute to project team]
```

**Schedule Trigger configuration:**
- Rule: `0 2 * * *` (daily at 02:00)
- Timezone: project-local timezone

---

## Common Operations

### Credential Configuration for AEC Services

| Service | n8n Credential Type | Header Name | Header Value |
|---------|-------------------|-------------|--------------|
| ERPNext | Header Auth | `Authorization` | `token {api_key}:{api_secret}` |
| Speckle | Header Auth | `Authorization` | `Bearer {personal_access_token}` |
| IfcPipeline | Header Auth | `X-API-Key` | `{api_key}` |

### HTTP Request to ERPNext API

```json
{
  "method": "POST",
  "url": "https://erp.example.com/api/resource/BOM",
  "authentication": "genericCredentialType",
  "genericAuthType": "httpHeaderAuth",
  "sendBody": true,
  "specifyBody": "json",
  "jsonBody": "={{ JSON.stringify({ data: { item: $json.item_code, items: $json.bom_items, quantity: 1 } }) }}"
}
```

### Execute Command for IfcOpenShell

```json
{
  "command": "python3 /scripts/extract_quantities.py --input \"{{ $json.file_path }}\" --format json",
  "timeout": 120000
}
```

**ALWAYS** set timeout to at least 120000ms (2 minutes) for IFC processing scripts. Large models require 5-10 minutes — set timeout accordingly.

### IfcPipeline Integration via HTTP Request

```json
{
  "method": "POST",
  "url": "http://ifcpipeline:8000/ifctester",
  "authentication": "genericCredentialType",
  "sendBody": true,
  "specifyBody": "json",
  "jsonBody": "={{ JSON.stringify({ ifc_file: $json.file_path, ids_file: '/specs/project.ids' }) }}"
}
```

**IfcPipeline endpoints:**

| Endpoint | Purpose | Input |
|----------|---------|-------|
| `/ifcconvert` | Convert IFC to GLB, OBJ, DAE | IFC file path + target format |
| `/ifccsv` | Export/import IFC data as CSV | IFC file path |
| `/ifcclash` | Geometric clash detection | Two IFC file paths |
| `/ifctester` | IDS validation | IFC file path + IDS file |
| `/ifcdiff` | Model version comparison | Two IFC file paths |
| `/calculate-qtos` | Quantity takeoff | IFC file path |
| `/ifc2json` | JSON export for viewers | IFC file path |

### Error Handling Pattern

ALWAYS add an Error Trigger workflow that catches failures from production pipelines:

```
[Error Trigger] → [Code: format error details]
    → [If: severity]
        ├── CRITICAL → [Send Email: ops team] + [HTTP Request: Slack alert]
        └── WARNING  → [Send Email: project lead]
```

Set `continueOnFail: true` on nodes where partial failure is acceptable (e.g., notification nodes).

### Sub-workflow Pattern for Reusable Stages

Extract common operations into sub-workflows called via Execute Sub-workflow node:

- **IFC Validation Sub-workflow**: Webhook input → validate → return pass/fail JSON
- **ERPNext BOM Creation Sub-workflow**: JSON input → create Items → create BOM → return BOM name
- **Notification Sub-workflow**: message + recipients → email + Slack

---

## n8n-nodes-ifcpipeline Community Node

The `n8n-nodes-ifcpipeline` package (install via n8n Community Nodes) provides dedicated nodes for IfcPipeline operations without manual HTTP configuration:

```bash
# Install in n8n (Settings → Community Nodes)
n8n-nodes-ifcpipeline
```

This replaces manual HTTP Request configuration for all `/ifc*` endpoints with visual node configuration.

---

## Reference Links

- [references/methods.md](references/methods.md) — n8n node reference, API endpoints, credential setup
- [references/examples.md](references/examples.md) — Complete workflow JSON snippets for AEC pipelines
- [references/anti-patterns.md](references/anti-patterns.md) — Common n8n AEC pipeline mistakes

### Official Sources

- https://docs.n8n.io/integrations/builtin/core-nodes/ — n8n core node reference
- https://docs.n8n.io/hosting/installation/server-setups/docker-compose/ — n8n Docker deployment
- https://docs.n8n.io/code/builtin/overview/ — n8n Code node reference
- https://speckle.guide/dev/server-webhooks.html — Speckle webhook documentation
- https://frappeframework.com/docs/user/en/api/rest — ERPNext/Frappe REST API
- https://github.com/jonatanjacobsson/ifcpipeline — IfcPipeline project
- https://github.com/jonatanjacobsson/n8n-nodes-ifcpipeline — n8n-nodes-ifcpipeline community node
