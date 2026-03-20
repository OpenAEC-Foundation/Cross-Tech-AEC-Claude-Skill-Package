# examples.md — Complete n8n Workflow JSON Snippets for AEC Pipelines

## Example 1: IFC Upload → Validation → ERPNext BOM Creation

This workflow receives an IFC file via webhook, validates it, extracts quantities, and creates a BOM in ERPNext.

```json
{
  "name": "IFC to ERPNext BOM Pipeline",
  "nodes": [
    {
      "name": "IFC Upload Webhook",
      "type": "n8n-nodes-base.webhook",
      "position": [250, 300],
      "parameters": {
        "httpMethod": "POST",
        "path": "ifc-upload",
        "binaryPropertyName": "ifc_file",
        "options": {
          "rawBody": true
        }
      },
      "typeVersion": 2
    },
    {
      "name": "Save IFC to Disk",
      "type": "n8n-nodes-base.executeCommand",
      "position": [450, 300],
      "parameters": {
        "command": "cp /tmp/n8n-binary-{{ $binary.ifc_file.fileName }} /data/models/{{ $binary.ifc_file.fileName }}"
      },
      "typeVersion": 1
    },
    {
      "name": "Validate IFC",
      "type": "n8n-nodes-base.executeCommand",
      "position": [650, 300],
      "parameters": {
        "command": "python3 /scripts/validate_ifc.py --input \"/data/models/{{ $binary.ifc_file.fileName }}\" --format json",
        "timeout": 120000
      },
      "typeVersion": 1
    },
    {
      "name": "Check Validation Result",
      "type": "n8n-nodes-base.if",
      "position": [850, 300],
      "parameters": {
        "conditions": {
          "options": { "caseSensitive": true },
          "combinator": "and",
          "conditions": [
            {
              "leftValue": "={{ $json.exitCode }}",
              "rightValue": 0,
              "operator": { "type": "number", "operation": "equals" }
            }
          ]
        }
      },
      "typeVersion": 2
    },
    {
      "name": "Extract Quantities",
      "type": "n8n-nodes-base.executeCommand",
      "position": [1050, 200],
      "parameters": {
        "command": "python3 /scripts/extract_quantities.py --input \"/data/models/{{ $binary.ifc_file.fileName }}\" --format json",
        "timeout": 300000
      },
      "typeVersion": 1
    },
    {
      "name": "Transform to ERPNext Format",
      "type": "n8n-nodes-base.code",
      "position": [1250, 200],
      "parameters": {
        "jsCode": "const raw = JSON.parse($input.first().json.stdout);\nconst bomItems = raw.elements.map(elem => ({\n  item_code: `${elem.ifc_class}_${elem.material}`.replace(/\\s+/g, '_'),\n  qty: elem.quantities.NetVolume || elem.quantities.NetArea || 1,\n  uom: elem.quantities.NetVolume ? 'm3' : (elem.quantities.NetArea ? 'm2' : 'Nos'),\n  rate: 0\n}));\n\nreturn [{ json: {\n  project_code: raw.project_name,\n  bom_items: bomItems\n}}];"
      },
      "typeVersion": 2
    },
    {
      "name": "Create ERPNext BOM",
      "type": "n8n-nodes-base.httpRequest",
      "position": [1450, 200],
      "parameters": {
        "method": "POST",
        "url": "https://erp.example.com/api/resource/BOM",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({ data: { item: $json.project_code, items: $json.bom_items, quantity: 1 } }) }}"
      },
      "credentials": {
        "httpHeaderAuth": { "id": "erpnext-token", "name": "ERPNext API Token" }
      },
      "typeVersion": 4
    },
    {
      "name": "Send Success Email",
      "type": "n8n-nodes-base.emailSend",
      "position": [1650, 200],
      "parameters": {
        "fromEmail": "bim-pipeline@example.com",
        "toEmail": "cost-manager@example.com",
        "subject": "BOM Created: {{ $json.data.name }}",
        "text": "A new BOM has been created from IFC model {{ $binary.ifc_file.fileName }}.\n\nBOM Name: {{ $json.data.name }}\nItems: {{ $json.data.items.length }}"
      },
      "typeVersion": 2
    },
    {
      "name": "Send Validation Failure Email",
      "type": "n8n-nodes-base.emailSend",
      "position": [1050, 400],
      "parameters": {
        "fromEmail": "bim-pipeline@example.com",
        "toEmail": "architect@example.com",
        "subject": "IFC Validation Failed: {{ $binary.ifc_file.fileName }}",
        "text": "The uploaded IFC file failed validation.\n\nFile: {{ $binary.ifc_file.fileName }}\nErrors:\n{{ $json.stderr }}"
      },
      "typeVersion": 2
    }
  ],
  "connections": {
    "IFC Upload Webhook": { "main": [[{ "node": "Save IFC to Disk", "type": "main", "index": 0 }]] },
    "Save IFC to Disk": { "main": [[{ "node": "Validate IFC", "type": "main", "index": 0 }]] },
    "Validate IFC": { "main": [[{ "node": "Check Validation Result", "type": "main", "index": 0 }]] },
    "Check Validation Result": {
      "main": [
        [{ "node": "Extract Quantities", "type": "main", "index": 0 }],
        [{ "node": "Send Validation Failure Email", "type": "main", "index": 0 }]
      ]
    },
    "Extract Quantities": { "main": [[{ "node": "Transform to ERPNext Format", "type": "main", "index": 0 }]] },
    "Transform to ERPNext Format": { "main": [[{ "node": "Create ERPNext BOM", "type": "main", "index": 0 }]] },
    "Create ERPNext BOM": { "main": [[{ "node": "Send Success Email", "type": "main", "index": 0 }]] }
  }
}
```

---

## Example 2: Speckle Commit Webhook → Change Detection → Slack Notification

This workflow listens for Speckle model commits and notifies the team of significant changes.

```json
{
  "name": "Speckle Change Notification",
  "nodes": [
    {
      "name": "Speckle Webhook",
      "type": "n8n-nodes-base.webhook",
      "position": [250, 300],
      "parameters": {
        "httpMethod": "POST",
        "path": "speckle-commit",
        "responseMode": "onReceived"
      },
      "typeVersion": 2
    },
    {
      "name": "Fetch Commit Details",
      "type": "n8n-nodes-base.httpRequest",
      "position": [450, 300],
      "parameters": {
        "method": "POST",
        "url": "https://speckle.example.com/graphql",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({ query: `{ stream(id: \"${$json.body.payload.streamId}\") { name commit(id: \"${$json.body.payload.commitId}\") { message authorName createdAt referencedObject branchName totalChildrenCount } } }` }) }}"
      },
      "credentials": {
        "httpHeaderAuth": { "id": "speckle-pat", "name": "Speckle PAT" }
      },
      "typeVersion": 4
    },
    {
      "name": "Evaluate Change Significance",
      "type": "n8n-nodes-base.code",
      "position": [650, 300],
      "parameters": {
        "jsCode": "const commit = $input.first().json.data.stream.commit;\nconst streamName = $input.first().json.data.stream.name;\n\n// Determine significance based on children count and branch\nconst isSignificant = commit.totalChildrenCount > 100 || commit.branchName === 'main';\n\nreturn [{ json: {\n  stream_name: streamName,\n  author: commit.authorName,\n  message: commit.message,\n  branch: commit.branchName,\n  object_count: commit.totalChildrenCount,\n  created_at: commit.createdAt,\n  is_significant: isSignificant\n}}];"
      },
      "typeVersion": 2
    },
    {
      "name": "Is Significant?",
      "type": "n8n-nodes-base.if",
      "position": [850, 300],
      "parameters": {
        "conditions": {
          "combinator": "and",
          "conditions": [
            {
              "leftValue": "={{ $json.is_significant }}",
              "rightValue": true,
              "operator": { "type": "boolean", "operation": "true" }
            }
          ]
        }
      },
      "typeVersion": 2
    },
    {
      "name": "Send Slack Notification",
      "type": "n8n-nodes-base.httpRequest",
      "position": [1050, 200],
      "parameters": {
        "method": "POST",
        "url": "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({ text: `*BIM Model Update*\\nStream: ${$json.stream_name}\\nBranch: ${$json.branch}\\nAuthor: ${$json.author}\\nMessage: ${$json.message}\\nObjects: ${$json.object_count}` }) }}"
      },
      "typeVersion": 4
    }
  ],
  "connections": {
    "Speckle Webhook": { "main": [[{ "node": "Fetch Commit Details", "type": "main", "index": 0 }]] },
    "Fetch Commit Details": { "main": [[{ "node": "Evaluate Change Significance", "type": "main", "index": 0 }]] },
    "Evaluate Change Significance": { "main": [[{ "node": "Is Significant?", "type": "main", "index": 0 }]] },
    "Is Significant?": {
      "main": [
        [{ "node": "Send Slack Notification", "type": "main", "index": 0 }],
        []
      ]
    }
  }
}
```

---

## Example 3: Nightly BIM Quality Check

This workflow runs a scheduled quality check against the latest IFC model and emails the results.

```json
{
  "name": "Nightly BIM Quality Check",
  "nodes": [
    {
      "name": "Schedule Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "position": [250, 300],
      "parameters": {
        "rule": {
          "interval": [{ "field": "cronExpression", "expression": "0 2 * * *" }]
        }
      },
      "typeVersion": 1
    },
    {
      "name": "Find Latest IFC",
      "type": "n8n-nodes-base.executeCommand",
      "position": [450, 300],
      "parameters": {
        "command": "ls -t /data/models/*.ifc | head -1"
      },
      "typeVersion": 1
    },
    {
      "name": "Run Quality Check",
      "type": "n8n-nodes-base.executeCommand",
      "position": [650, 300],
      "parameters": {
        "command": "python3 /scripts/bim_quality_check.py --input \"{{ $json.stdout.trim() }}\" --ids /specs/project.ids --format json",
        "timeout": 600000
      },
      "typeVersion": 1
    },
    {
      "name": "Format HTML Report",
      "type": "n8n-nodes-base.code",
      "position": [850, 300],
      "parameters": {
        "jsCode": "const results = JSON.parse($input.first().json.stdout);\nlet html = '<h2>BIM Quality Report</h2>';\nhtml += `<p>Model: ${results.model_name}</p>`;\nhtml += `<p>Date: ${new Date().toISOString().split('T')[0]}</p>`;\nhtml += `<p>Total checks: ${results.total_checks}</p>`;\nhtml += `<p>Passed: ${results.passed} | Failed: ${results.failed}</p>`;\n\nif (results.failures && results.failures.length > 0) {\n  html += '<h3>Failures</h3><table border=\"1\"><tr><th>Element</th><th>Rule</th><th>Detail</th></tr>';\n  results.failures.forEach(f => {\n    html += `<tr><td>${f.element}</td><td>${f.rule}</td><td>${f.detail}</td></tr>`;\n  });\n  html += '</table>';\n}\n\nreturn [{ json: { html_report: html, passed: results.passed, failed: results.failed, model: results.model_name } }];"
      },
      "typeVersion": 2
    },
    {
      "name": "Send Report Email",
      "type": "n8n-nodes-base.emailSend",
      "position": [1050, 300],
      "parameters": {
        "fromEmail": "bim-pipeline@example.com",
        "toEmail": "project-team@example.com",
        "subject": "BIM Quality Report: {{ $json.passed }}/{{ $json.passed + $json.failed }} passed — {{ $json.model }}",
        "html": "={{ $json.html_report }}"
      },
      "typeVersion": 2
    }
  ],
  "connections": {
    "Schedule Trigger": { "main": [[{ "node": "Find Latest IFC", "type": "main", "index": 0 }]] },
    "Find Latest IFC": { "main": [[{ "node": "Run Quality Check", "type": "main", "index": 0 }]] },
    "Run Quality Check": { "main": [[{ "node": "Format HTML Report", "type": "main", "index": 0 }]] },
    "Format HTML Report": { "main": [[{ "node": "Send Report Email", "type": "main", "index": 0 }]] }
  }
}
```

---

## Example 4: IfcPipeline Clash Detection Workflow

This workflow uses IfcPipeline's REST API to run clash detection between architectural and structural models.

```json
{
  "name": "IFC Clash Detection",
  "nodes": [
    {
      "name": "Manual Trigger",
      "type": "n8n-nodes-base.manualTrigger",
      "position": [250, 300],
      "parameters": {},
      "typeVersion": 1
    },
    {
      "name": "Run Clash Detection",
      "type": "n8n-nodes-base.httpRequest",
      "position": [450, 300],
      "parameters": {
        "method": "POST",
        "url": "http://ifcpipeline:8000/ifcclash",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "{\"ifc_file_a\": \"/data/models/architectural.ifc\", \"ifc_file_b\": \"/data/models/structural.ifc\", \"tolerance\": 0.01}",
        "timeout": 600000
      },
      "credentials": {
        "httpHeaderAuth": { "id": "ifcpipeline-key", "name": "IfcPipeline API Key" }
      },
      "typeVersion": 4
    },
    {
      "name": "Check Clashes Found",
      "type": "n8n-nodes-base.if",
      "position": [650, 300],
      "parameters": {
        "conditions": {
          "combinator": "and",
          "conditions": [
            {
              "leftValue": "={{ $json.clash_count }}",
              "rightValue": 0,
              "operator": { "type": "number", "operation": "gt" }
            }
          ]
        }
      },
      "typeVersion": 2
    },
    {
      "name": "Format Clash Report",
      "type": "n8n-nodes-base.code",
      "position": [850, 200],
      "parameters": {
        "jsCode": "const clashes = $input.first().json;\nlet report = `Clash Detection Report\\n${'='.repeat(40)}\\n`;\nreport += `Architectural: architectural.ifc\\n`;\nreport += `Structural: structural.ifc\\n`;\nreport += `Total clashes: ${clashes.clash_count}\\n\\n`;\n\nclashes.clashes.slice(0, 50).forEach((c, i) => {\n  report += `${i+1}. ${c.element_a} ↔ ${c.element_b} (distance: ${c.distance.toFixed(3)}m)\\n`;\n});\n\nreturn [{ json: { report, clash_count: clashes.clash_count } }];"
      },
      "typeVersion": 2
    },
    {
      "name": "Notify Team",
      "type": "n8n-nodes-base.emailSend",
      "position": [1050, 200],
      "parameters": {
        "fromEmail": "bim-pipeline@example.com",
        "toEmail": "coordination@example.com",
        "subject": "⚠ {{ $json.clash_count }} clashes detected between Arch/Struct models",
        "text": "={{ $json.report }}"
      },
      "typeVersion": 2
    }
  ],
  "connections": {
    "Manual Trigger": { "main": [[{ "node": "Run Clash Detection", "type": "main", "index": 0 }]] },
    "Run Clash Detection": { "main": [[{ "node": "Check Clashes Found", "type": "main", "index": 0 }]] },
    "Check Clashes Found": {
      "main": [
        [{ "node": "Format Clash Report", "type": "main", "index": 0 }],
        []
      ]
    },
    "Format Clash Report": { "main": [[{ "node": "Notify Team", "type": "main", "index": 0 }]] }
  }
}
```

---

## Example 5: Error Handling Workflow (Error Trigger)

This workflow catches unhandled errors from ANY production pipeline and routes alerts by severity.

```json
{
  "name": "AEC Pipeline Error Handler",
  "nodes": [
    {
      "name": "Error Trigger",
      "type": "n8n-nodes-base.errorTrigger",
      "position": [250, 300],
      "parameters": {},
      "typeVersion": 1
    },
    {
      "name": "Format Error Details",
      "type": "n8n-nodes-base.code",
      "position": [450, 300],
      "parameters": {
        "jsCode": "const error = $input.first().json;\nconst workflowName = error.workflow?.name || 'Unknown';\nconst nodeName = error.execution?.lastNodeExecuted || 'Unknown';\nconst errorMsg = error.execution?.error?.message || 'No message';\n\n// Classify severity\nlet severity = 'WARNING';\nif (nodeName.includes('ERPNext') || nodeName.includes('BOM')) severity = 'CRITICAL';\nif (nodeName.includes('Email') || nodeName.includes('Slack')) severity = 'INFO';\n\nreturn [{ json: {\n  severity,\n  workflow: workflowName,\n  node: nodeName,\n  error: errorMsg,\n  timestamp: new Date().toISOString()\n}}];"
      },
      "typeVersion": 2
    },
    {
      "name": "Route by Severity",
      "type": "n8n-nodes-base.switch",
      "position": [650, 300],
      "parameters": {
        "dataType": "string",
        "value1": "={{ $json.severity }}",
        "rules": {
          "rules": [
            { "value2": "CRITICAL" },
            { "value2": "WARNING" },
            { "value2": "INFO" }
          ]
        }
      },
      "typeVersion": 3
    },
    {
      "name": "Alert Ops Team",
      "type": "n8n-nodes-base.emailSend",
      "position": [850, 150],
      "parameters": {
        "fromEmail": "bim-pipeline@example.com",
        "toEmail": "ops@example.com",
        "subject": "CRITICAL: {{ $json.workflow }} failed at {{ $json.node }}",
        "text": "Workflow: {{ $json.workflow }}\nNode: {{ $json.node }}\nError: {{ $json.error }}\nTime: {{ $json.timestamp }}"
      },
      "typeVersion": 2
    },
    {
      "name": "Alert Project Lead",
      "type": "n8n-nodes-base.emailSend",
      "position": [850, 300],
      "parameters": {
        "fromEmail": "bim-pipeline@example.com",
        "toEmail": "project-lead@example.com",
        "subject": "WARNING: {{ $json.workflow }} issue at {{ $json.node }}",
        "text": "Workflow: {{ $json.workflow }}\nNode: {{ $json.node }}\nError: {{ $json.error }}\nTime: {{ $json.timestamp }}"
      },
      "typeVersion": 2
    }
  ],
  "connections": {
    "Error Trigger": { "main": [[{ "node": "Format Error Details", "type": "main", "index": 0 }]] },
    "Format Error Details": { "main": [[{ "node": "Route by Severity", "type": "main", "index": 0 }]] },
    "Route by Severity": {
      "main": [
        [{ "node": "Alert Ops Team", "type": "main", "index": 0 }],
        [{ "node": "Alert Project Lead", "type": "main", "index": 0 }],
        []
      ]
    }
  }
}
```
