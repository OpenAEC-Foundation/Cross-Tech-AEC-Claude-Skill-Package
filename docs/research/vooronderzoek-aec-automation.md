# Vooronderzoek: AEC Automation — n8n Pipelines, Docker Stacks, ERPNext Costing, FreeCAD IFC

> Research document for the Cross-Technology AEC Integration Claude Skill Package.
> Covers four automation domains: IFC-to-ERPNext costing, FreeCAD-IFC bridging, n8n pipeline orchestration, and Docker-based AEC service stacks.

---

## 1. IFC-to-ERPNext Costing Integration

### 1.1 IFC Quantity Extraction with IfcOpenShell

The IFC schema defines quantities through the `IfcElementQuantity` entity, which groups individual quantity measures. IfcOpenShell 0.8.x provides the `ifcopenshell.util.element.get_psets()` utility as the primary API for accessing both property sets and quantity sets.

**Quantity types defined in IFC4:**

| IFC Type | Measures | Typical Use |
|----------|----------|-------------|
| `IfcQuantityArea` | m2 | Wall surfaces, floor areas, roof areas |
| `IfcQuantityVolume` | m3 | Concrete volumes, excavation volumes |
| `IfcQuantityLength` | m | Beam lengths, pipe runs, cable trays |
| `IfcQuantityCount` | integer | Door count, window count, fixture count |
| `IfcQuantityWeight` | kg | Steel tonnage, rebar weight |

**Extraction code pattern (IfcOpenShell 0.8.x, Python 3.10+):**

```python
import ifcopenshell
import ifcopenshell.util.element

model = ifcopenshell.open("project.ifc")

for wall in model.by_type("IfcWall"):
    # Get ALL property sets and quantity sets
    all_psets = ifcopenshell.util.element.get_psets(wall)
    # Returns: {"Pset_WallCommon": {"FireRating": "2HR", ...}, "Qto_WallBaseQuantities": {"Length": 5.0, ...}}

    # Get ONLY quantity sets (IfcElementQuantity)
    qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)
    # Returns: {"Qto_WallBaseQuantities": {"Length": 5.0, "Height": 3.0, "Width": 0.2, ...}}

    # Get ONLY property sets (IfcPropertySet)
    psets = ifcopenshell.util.element.get_psets(wall, psets_only=True)
    # Returns: {"Pset_WallCommon": {"FireRating": "2HR", "IsExternal": True, ...}}
```

**Critical detail:** `get_psets()` returns a flat dictionary where each key is the property/quantity set name and each value is a dictionary of properties. The special key `"id"` in each sub-dictionary contains the IfcOpenShell entity ID.

**Including type-level properties:**

```python
# Wall instance quantities (directly assigned)
instance_qtos = ifcopenshell.util.element.get_psets(wall, qtos_only=True)

# Wall type properties (inherited from IfcWallType)
wall_type = ifcopenshell.util.element.get_type(wall)
if wall_type:
    type_psets = ifcopenshell.util.element.get_psets(wall_type)
```

**Classification access for mapping to ERPNext Item Groups:**

```python
import ifcopenshell.util.classification

# Get classification references (Uniclass, OmniClass, etc.)
refs = ifcopenshell.util.classification.get_references(wall)
for ref in refs:
    print(ref.Identification)  # e.g., "Ss_25_10_30" (Uniclass)
    print(ref.Name)            # e.g., "Concrete wall systems"
```

### 1.2 ERPNext BOM Structure

ERPNext organizes manufacturing data around three core DocTypes:

**Item** — the master record for any stockable or non-stockable entity:
- `item_code`: unique identifier (maps from IFC element type + classification)
- `item_group`: hierarchical categorization (maps from IfcClassificationReference)
- `uom`: unit of measure (maps from IFC quantity unit)
- `description`: human-readable (maps from IFC element Name + Description)

**BOM (Bill of Materials)** — defines what materials compose a finished item:
- `item`: the finished product this BOM describes
- `items`: child table of component items with quantities
- `rm_cost_as_per`: cost calculation method ("Valuation Rate", "Last Purchase Rate", "Price List")
- Multi-level BOMs: a BOM item can reference another BOM (sub-assembly)
- Once submitted, a BOM is immutable — must cancel and recreate to modify

**BOM Item** (child row within a BOM):
- `item_code`: component material
- `qty`: quantity required
- `uom`: unit of measure
- `rate`: unit cost

### 1.3 Frappe/ERPNext REST API for Programmatic BOM Creation

ERPNext exposes a full REST API at `/api/resource/:doctype`. Authentication uses token-based headers.

**Creating an Item:**

```python
import requests

ERP_URL = "https://erp.example.com"
HEADERS = {
    "Authorization": "token api_key:api_secret",
    "Content-Type": "application/json"
}

# Create a new Item from IFC data
item_data = {
    "doctype": "Item",
    "item_code": "CONC-WALL-200MM",
    "item_name": "Concrete Wall 200mm",
    "item_group": "Raw Material",
    "stock_uom": "Cubic Meter",
    "description": "200mm concrete wall system (from IFC IfcWall)"
}

response = requests.post(
    f"{ERP_URL}/api/resource/Item",
    json=item_data,
    headers=HEADERS
)
```

**Creating a BOM from IFC quantities:**

```python
# Create BOM linking IFC quantities to ERPNext items
bom_data = {
    "doctype": "BOM",
    "item": "BUILDING-LEVEL-01",
    "items": [
        {
            "item_code": "CONC-WALL-200MM",
            "qty": 45.6,  # from Qto_WallBaseQuantities.GrossVolume
            "uom": "Cubic Meter"
        },
        {
            "item_code": "REBAR-B500B",
            "qty": 3200,  # from Qto_ReinforcingElementBaseQuantities.Weight
            "uom": "Kg"
        }
    ],
    "rm_cost_as_per": "Valuation Rate"
}

response = requests.post(
    f"{ERP_URL}/api/resource/BOM",
    json=bom_data,
    headers=HEADERS
)

# Submit the BOM (makes it active)
bom_name = response.json()["data"]["name"]
requests.put(
    f"{ERP_URL}/api/resource/BOM/{bom_name}",
    json={"docstatus": 1},
    headers=HEADERS
)
```

**CRITICAL:** The BOM endpoint is case-sensitive — use `/api/resource/BOM`, NOT `/api/resource/bom`.

### 1.4 Mapping Strategy: IFC to ERPNext

| IFC Concept | ERPNext Equivalent | Mapping Logic |
|-------------|-------------------|---------------|
| `IfcClassificationReference.Identification` | `Item.item_group` | Uniclass/OmniClass code to Item Group hierarchy |
| `IfcWallType.Name` | `Item.item_code` | Type name becomes item identifier |
| `IfcElementQuantity` quantities | `BOM Item.qty` | Direct quantity mapping with unit conversion |
| `IfcProject.Name` | `Project.project_name` | Top-level project mapping |
| `IfcBuildingStorey.Name` | `Cost Center` | Storey-level cost tracking |
| IFC unit (from `IfcUnitAssignment`) | `Item.stock_uom` | SI unit to ERPNext UOM name |

### 1.5 Data Loss at the Boundary

**IFC data with NO ERPNext equivalent:**
- Geometric representations (IfcShapeRepresentation) — ERPNext has no geometry model
- Spatial relationships (IfcRelContainedInSpatialStructure) — ERPNext uses flat item lists
- Material layer sets (IfcMaterialLayerSetUsage) — ERPNext tracks single-material items
- Connection geometry (IfcRelConnectsPathElements) — no structural analysis in ERPNext

**ERPNext data with NO IFC equivalent:**
- Pricing and valuation rates — IFC has no cost schema (IFC5 may add this)
- Warehouse and bin locations — IFC tracks building locations, not stock locations
- Supplier information — IFC has `IfcOrganization` but not procurement workflows
- Serial numbers and batch tracking — no equivalent in IFC

---

## 2. FreeCAD-IFC Bridge

### 2.1 FreeCAD BIM Workbench and NativeIFC

FreeCAD 1.0+ introduced the BIM Workbench as the primary tool for architectural modeling, replacing the older Arch Workbench. The BIM Workbench provides parametric BIM objects: walls, columns, beams, slabs, doors, windows, pipes, stairs, roofs, panels, and equipment.

The most significant development is **NativeIFC mode**, which fundamentally changes how FreeCAD handles IFC files. Instead of translating IFC entities into FreeCAD's internal Part representation (which causes data loss), NativeIFC treats the IFC file itself as the data structure.

### 2.2 NativeIFC: Two Operating Modes

**Locked Mode:**
- The FreeCAD document IS the IFC file — no separate `.FCStd` file
- All objects automatically convert to IFC format
- Modifications write directly to the IFC file
- Minimal file changes on save — modifying one element changes only that element's data line
- Suitable for Git-based version control of IFC files

**Unlocked Mode:**
- IFC and non-IFC elements coexist in one document
- IFC projects attach to separate IFC files
- More flexibility during early design stages
- Traditional file exports remain available

### 2.3 IfcOpenShell as FreeCAD's IFC Engine

FreeCAD uses IfcOpenShell as its underlying IFC engine — the same library used by BlenderBIM/Bonsai. This means:

- IFC parsing and writing use the same IfcOpenShell API documented in Section 1
- Geometry conversion uses IfcOpenShell's OpenCASCADE-based geometry kernel
- Property set and quantity access use `ifcopenshell.util.element.get_psets()`
- The NativeIFC initiative aims to upstream improvements to IfcOpenShell, benefiting both FreeCAD and Blender

### 2.4 Import Workflow: IFC to FreeCAD

**Supported entities on import:**
- `IfcWall`, `IfcColumn`, `IfcBeam`, `IfcSlab`, `IfcDoor`, `IfcWindow`, `IfcStair`, `IfcRoof`
- `IfcBuildingElementProxy` (generic elements)
- `IfcSpace` (room volumes)
- Spatial hierarchy: `IfcProject` → `IfcSite` → `IfcBuilding` → `IfcBuildingStorey`

**Geometry conversion:**
IFC geometry (CSG, Brep, swept solids) is converted to FreeCAD's Part::TopoShape via IfcOpenShell's OpenCASCADE binding. This conversion is lossless for Brep geometry but may approximate complex parametric definitions.

**NativeIFC import:**
In NativeIFC mode, geometry is NOT converted to FreeCAD's internal format. Instead, shapes are loaded on demand — you can load/unload geometry per object to manage memory for large models (100MB-2GB IFC files).

### 2.5 Export Workflow: FreeCAD to IFC

**What is preserved:**
- BIM object types (walls, columns, beams mapped to corresponding IfcXxx types)
- Property sets created in FreeCAD
- Material assignments
- Spatial hierarchy (sites, buildings, storeys)

**What is generated:**
- `IfcOwnerHistory` (created automatically)
- `IfcGeometricRepresentationContext`
- GUIDs for all entities

**What is lost:**
- FreeCAD-specific parametric constraints (Sketcher, Part constraints)
- Non-BIM objects (Part, Mesh) unless explicitly converted
- Draft/annotation objects

### 2.6 Round-Trip Editing

NativeIFC enables true round-trip editing:

1. Open IFC file in FreeCAD (locked mode)
2. Modify elements: change wall height, move objects, edit properties
3. Save — only changed elements are rewritten
4. The IFC file diff shows minimal changes, suitable for Git tracking

**Supported modifications in NativeIFC:**
- Change IFC class (wall to column, requires compatible geometry)
- Edit attributes: Name, Description, ObjectType
- Modify geometric properties: Height, Width, Length
- Drag objects between spatial containers
- Delete objects (removes from IFC file)
- Add/edit property sets and custom properties

**Limitations (FreeCAD 1.0):**
- Graphical editing (drag-move, stretch) is limited
- Not all IFC property types are editable
- Complex class conversions require intermediate steps
- Full shape loading required for geometry-dependent operations

### 2.7 FreeCAD Scripting API for IFC

```python
import FreeCAD
import Arch

# Open an IFC file in NativeIFC mode
doc = FreeCAD.openDocument("project.ifc")

# Access IFC objects
for obj in doc.Objects:
    if hasattr(obj, "IfcType"):
        print(f"{obj.Name}: {obj.IfcType}")

# Create a wall using BIM/Arch module
wall = Arch.makeWall(length=5000, width=200, height=3000)

# Set IFC properties
wall.IfcType = "IfcWall"
wall.Label = "Exterior Wall 01"

# Access IFC data via IfcOpenShell (NativeIFC)
import ifcopenshell
ifc_file = ifcopenshell.open("project.ifc")
walls = ifc_file.by_type("IfcWall")

# Export to IFC
import importIFC
importIFC.export([wall], "output.ifc")

# Recompute and save
doc.recompute()
doc.save()
```

---

## 3. n8n Pipeline Automation for AEC

### 3.1 n8n Workflow Engine Overview

n8n (v1.x, self-hostable, source-available license) is a visual workflow automation platform. Workflows consist of nodes connected by edges, where data flows as JSON arrays from node to node.

**Core nodes relevant to AEC automation:**

| Node | Exact Name | Purpose in AEC |
|------|-----------|----------------|
| HTTP Request | `n8n-nodes-base.httpRequest` | Call REST APIs (ERPNext, Speckle, custom services) |
| Webhook | `n8n-nodes-base.webhook` | Receive IFC upload events, Speckle commit hooks |
| Code | `n8n-nodes-base.code` | Transform IFC data, parse JSON, compute quantities |
| Execute Command | `n8n-nodes-base.executeCommand` | Run IfcOpenShell Python scripts, CLI tools |
| Schedule Trigger | `n8n-nodes-base.scheduleTrigger` | Nightly BIM quality checks, scheduled exports |
| FTP | `n8n-nodes-base.ftp` | Upload/download IFC files from project servers |
| If | `n8n-nodes-base.if` | Route based on validation results |
| Switch | `n8n-nodes-base.switch` | Route by IFC schema version or element type |
| Merge | `n8n-nodes-base.merge` | Combine data from multiple AEC sources |
| Execute Sub-workflow | `n8n-nodes-base.executeWorkflow` | Modular pipeline stages |

### 3.2 AEC Pipeline Patterns

**Pattern 1: IFC Upload → Validation → Report**

```json
{
  "nodes": [
    {
      "name": "IFC Upload Webhook",
      "type": "n8n-nodes-base.webhook",
      "parameters": {
        "httpMethod": "POST",
        "path": "ifc-upload",
        "binaryPropertyName": "ifc_file",
        "options": { "rawBody": true }
      }
    },
    {
      "name": "Validate IFC",
      "type": "n8n-nodes-base.executeCommand",
      "parameters": {
        "command": "python3 /scripts/validate_ifc.py --input /tmp/{{ $binary.ifc_file.fileName }}"
      }
    },
    {
      "name": "Extract Quantities",
      "type": "n8n-nodes-base.executeCommand",
      "parameters": {
        "command": "python3 /scripts/extract_quantities.py --input /tmp/{{ $binary.ifc_file.fileName }} --output /tmp/quantities.json"
      }
    },
    {
      "name": "Create ERPNext BOM",
      "type": "n8n-nodes-base.httpRequest",
      "parameters": {
        "method": "POST",
        "url": "https://erp.example.com/api/resource/BOM",
        "authentication": "genericCredentialType",
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            { "name": "item", "value": "={{ $json.project_code }}" },
            { "name": "items", "value": "={{ $json.bom_items }}" }
          ]
        }
      }
    }
  ]
}
```

**Pattern 2: Speckle Commit → Change Detection → Notification**

This pattern uses Speckle's webhook system. When a new commit is pushed to a Speckle stream, a webhook triggers the n8n workflow:

1. **Webhook node** receives the Speckle commit event (stream ID, commit ID, message)
2. **HTTP Request node** fetches commit details from Speckle GraphQL API
3. **Code node** compares with previous commit to identify changed elements
4. **If node** routes based on change severity
5. **HTTP Request node** posts notification to Slack/Teams/email

**Pattern 3: Scheduled BIM Quality Check**

1. **Schedule Trigger** fires nightly at 02:00
2. **Execute Command** runs IfcOpenShell-based quality checks on latest model
3. **Code node** formats results into HTML report
4. **Send Email node** distributes report to project team

### 3.3 IfcPipeline: Dedicated n8n-to-IFC Integration

The IfcPipeline project (https://github.com/jonatanjacobsson/ifcpipeline) provides a FastAPI-based microservice specifically designed for IFC processing with n8n integration:

**Architecture:**
- FastAPI gateway with REST endpoints
- Specialized worker containers for each operation
- Redis Queue (RQ) for async job management
- PostgreSQL for result persistence
- Community n8n node package: `n8n-nodes-ifcpipeline`

**Available endpoints:**
- `/ifcconvert` — convert IFC to GLB, STEP, DAE, OBJ, XML
- `/ifccsv` — bidirectional CSV/XLSX export-import
- `/ifcclash` — geometric clash detection
- `/ifctester` — IDS (Information Delivery Specification) validation
- `/ifcdiff` — model version comparison
- `/calculate-qtos` — automated quantity takeoff (5D BIM)
- `/patch/execute` — apply modification recipes to IFC files
- `/ifc2json` — JSON export for web viewers

The `n8n-nodes-ifcpipeline` community node package enables visual orchestration of these operations within n8n workflows without writing code.

### 3.4 n8n Credential Management for AEC Services

n8n stores credentials encrypted in its database. For AEC integration, configure these credential types:

| Service | Credential Type | Required Fields |
|---------|----------------|-----------------|
| ERPNext | Header Auth | `Authorization: token api_key:api_secret` |
| Speckle | Header Auth | `Authorization: Bearer <personal_access_token>` |
| IfcPipeline | Header Auth | `X-API-Key: <api_key>` |
| QGIS Server | None (typically unauthenticated on internal network) | — |

---

## 4. Docker-Based AEC Service Stack

### 4.1 Containerizing IfcOpenShell

IfcOpenShell requires OpenCASCADE Technology (OCCT) for geometry processing. The recommended approach is conda-based installation due to binary dependency management.

**Dockerfile for IfcOpenShell worker:**

```dockerfile
FROM continuumio/miniconda3:24.7.1-0

# Install IfcOpenShell with geometry support via conda
RUN conda install -c conda-forge ifcopenshell pythonocc-core -y && \
    conda clean -afy

# Install additional Python dependencies
RUN pip install --no-cache-dir \
    fastapi==0.115.0 \
    uvicorn==0.30.0 \
    redis==5.0.0 \
    rq==1.16.0

WORKDIR /app
COPY ./scripts /app/scripts

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Resource requirements:**
- Base image: ~400MB (miniconda3)
- IfcOpenShell + OCCT: ~600MB additional
- Runtime memory: 2-4GB for processing large IFC files (100MB-2GB)
- CPU: geometry operations are single-threaded per model, multi-model parallelism via workers

**Alternative: community Docker image:**
- `aecgeeks/ifcopenshell` — pre-built image with IfcOpenShell (available on Docker Hub, but verify tag currency before use)

### 4.2 Speckle Server Docker Deployment

Speckle Server requires six services plus three infrastructure dependencies. The official deployment uses two docker-compose files: `docker-compose-deps.yml` for infrastructure and `docker-compose-speckle.yml` for application services.

**Infrastructure services (docker-compose-deps.yml):**

| Service | Image | Purpose | Resources |
|---------|-------|---------|-----------|
| PostgreSQL | `postgres:14` | Primary database | 1GB+ RAM |
| Redis | `redis:7-alpine` | Caching, job queues | 256MB RAM |
| MinIO | `minio/minio` | S3-compatible blob storage | 512MB RAM, disk for IFC files |

**Application services (docker-compose-speckle.yml):**

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| `speckle-server` | `speckle/speckle-server` | 3000 | Core API (GraphQL + REST) |
| `speckle-frontend-2` | `speckle/speckle-frontend-2` | — | Nuxt.js web frontend |
| `speckle-ingress` | `speckle/speckle-docker-compose-ingress` | 80 | Nginx reverse proxy |
| `preview-service` | `speckle/speckle-preview-service` | 3001 | 3D preview generation |
| `webhook-service` | `speckle/speckle-webhook-service` | — | Outbound webhook delivery |
| `ifc-import-server` | `speckle/ifc-import-service` | — | IFC file import processing |

**Minimum server requirements:** 4GB RAM, 2 CPU cores, 20GB disk (more for large IFC file storage).

**Critical environment variables for `speckle-server`:**

```yaml
CANONICAL_URL: "https://speckle.example.com"
SESSION_SECRET: "<generate-random-secret>"  # NEVER use default
STRATEGY_LOCAL: "true"  # Enable username/password auth
POSTGRES_URL: "postgres://speckle:speckle@postgres/speckle"
REDIS_URL: "redis://redis"
S3_ENDPOINT: "http://minio:9000"
S3_ACCESS_KEY: "<minio-access-key>"
S3_SECRET_KEY: "<minio-secret-key>"
FILE_SIZE_LIMIT_MB: 1000  # IFC files can be large
```

All Speckle images use `platform: linux/amd64` and `restart: always`.

### 4.3 QGIS Server in Docker

QGIS Server provides OGC-compliant WMS/WFS/WCS endpoints for serving geospatial data. The official Docker setup uses the `qgis/qgis-server` image.

**Basic deployment:**

```yaml
qgis-server:
  image: qgis/qgis-server:ltr
  ports:
    - "8010:80"
  volumes:
    - ./qgis-projects:/project:ro
    - ./qgis-data:/data:ro
  environment:
    QGIS_PROJECT_FILE: /project/project.qgs
```

**OGC endpoint patterns:**
- WMS GetMap: `http://localhost:8010/ogc/project?SERVICE=WMS&VERSION=1.3.0&REQUEST=GetMap&LAYERS=buildings`
- WFS GetFeature: `http://localhost:8010/ogc/project?SERVICE=WFS&VERSION=2.0.0&REQUEST=GetFeature&TYPENAME=parcels`
- WFS3/OGC API Features: `http://localhost:8010/wfs3/`

**Alternative image:** `kartoza/qgis-server` provides additional configuration options and is widely used in production deployments.

**BIM-GIS integration use case:** Load IFC site coordinates, transform to a geographic CRS (e.g., EPSG:28992 for the Netherlands), and serve georeferenced building footprints as WFS layers that other GIS tools can consume.

### 4.4 web-ifc Processing Workers

web-ifc (by ThatOpen/IFC.js) enables server-side IFC processing in Node.js using WebAssembly. The library includes dedicated Node.js bindings: `web-ifc-api-node.js` and `web-ifc-node.wasm`.

**Dockerfile for web-ifc worker:**

```dockerfile
FROM node:20-slim

WORKDIR /app

# Install web-ifc
RUN npm init -y && \
    npm install web-ifc@0.0.57

COPY ./src /app/src

EXPOSE 3000
CMD ["node", "src/server.js"]
```

**Server-side IFC processing example:**

```javascript
const WebIFC = require("web-ifc/web-ifc-api-node.js");

const ifcApi = new WebIFC.IfcAPI();
await ifcApi.Init();

// Load IFC file from buffer
const data = fs.readFileSync("/data/model.ifc");
const modelID = ifcApi.OpenModel(data);

// Get all IfcWall entities
const walls = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCWALL);
console.log(`Found ${walls.size()} walls`);

// Extract geometry for Three.js viewer
const meshData = ifcApi.GetGeometry(modelID, walls.get(0));

ifcApi.CloseModel(modelID);
```

**Resource requirements:**
- Image size: ~200MB (node:20-slim + web-ifc WASM)
- Memory: 1-2GB for medium IFC files, up to 4GB for large models
- web-ifc is faster than IfcOpenShell for pure geometry extraction but has less comprehensive property/relationship APIs

### 4.5 Multi-Service docker-compose for AEC Development

The following docker-compose combines the core AEC services into a single development stack:

```yaml
version: "3.8"

services:
  # ============================================
  # Infrastructure
  # ============================================
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: aec
      POSTGRES_PASSWORD: aec_dev_password
      POSTGRES_DB: aec
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"
      - "9001:9001"

  # ============================================
  # AEC Processing Services
  # ============================================
  ifcopenshell-worker:
    build:
      context: ./services/ifcopenshell
      dockerfile: Dockerfile
    volumes:
      - ifc_data:/data/ifc
      - ./scripts:/app/scripts:ro
    environment:
      REDIS_URL: redis://redis:6379
      POSTGRES_URL: postgresql://aec:aec_dev_password@postgres/aec
    depends_on:
      - redis
      - postgres
    deploy:
      resources:
        limits:
          memory: 4G

  web-ifc-worker:
    build:
      context: ./services/web-ifc
      dockerfile: Dockerfile
    volumes:
      - ifc_data:/data/ifc:ro
    ports:
      - "3001:3000"
    deploy:
      resources:
        limits:
          memory: 2G

  qgis-server:
    image: qgis/qgis-server:ltr
    volumes:
      - ./qgis-projects:/project:ro
      - ./gis-data:/data:ro
    ports:
      - "8010:80"

  # ============================================
  # Workflow Automation
  # ============================================
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      N8N_BASIC_AUTH_ACTIVE: "true"
      N8N_BASIC_AUTH_USER: admin
      N8N_BASIC_AUTH_PASSWORD: change_me
      EXECUTIONS_MODE: queue
      QUEUE_BULL_REDIS_HOST: redis
      GENERIC_TIMEZONE: Europe/Amsterdam
    volumes:
      - n8n_data:/home/node/.n8n
      - ifc_data:/data/ifc
    depends_on:
      - redis

  # ============================================
  # Business Logic
  # ============================================
  erpnext:
    # ERPNext requires its own compose stack — reference only
    # See: https://github.com/frappe/frappe_docker
    # Typically runs on separate infrastructure
    # Connect via REST API from n8n
    image: frappe/erpnext:v15
    # Full ERPNext deployment requires:
    # frappe/erpnext (web), redis-cache, redis-queue,
    # redis-socketio, mariadb:10.6, frappe/bench-worker
    profiles:
      - erpnext  # Only start with --profile erpnext

volumes:
  postgres_data:
  minio_data:
  n8n_data:
  ifc_data:
```

### 4.6 Volume Management for Large IFC Files

IFC files in real AEC projects range from 50MB (single discipline) to 2GB+ (federated models). Volume management strategies:

**Shared named volume (`ifc_data`):**
- Mounted by ifcopenshell-worker, web-ifc-worker, and n8n
- Enables file-based communication between services
- Use sub-directories per project: `/data/ifc/{project_id}/{filename}.ifc`

**Object storage (MinIO/S3):**
- Better for multi-node deployments where named volumes do not span hosts
- Upload via presigned URLs, download on demand
- Use for archival; copy to local volume for processing

**Tmpfs for processing:**
- For ephemeral intermediate files (converted formats, extracted data)
- Faster I/O, auto-cleaned on container restart

### 4.7 Network Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │              Docker Network (aec-net)        │
                    │                                             │
  External          │  ┌──────────┐    ┌───────────────────┐     │
  API calls  ──────►│  │   n8n    │───►│ ifcopenshell-worker│     │
  (port 5678)       │  │ :5678    │    │ :8000 (internal)  │     │
                    │  └────┬─────┘    └───────────────────┘     │
                    │       │                                     │
                    │       ├──────────►┌──────────────────┐     │
                    │       │           │  web-ifc-worker   │     │
                    │       │           │  :3000 (internal) │     │
                    │       │           └──────────────────┘     │
                    │       │                                     │
  WMS/WFS    ──────►│  ┌────▼─────┐    ┌──────────────────┐     │
  (port 8010)       │  │  qgis    │    │    ERPNext        │     │
                    │  │  server  │    │  (external/API)   │     │
                    │  └──────────┘    └──────────────────┘     │
                    │                                             │
                    │  ┌──────────┐ ┌───────┐ ┌───────┐         │
                    │  │ postgres │ │ redis │ │ minio │         │
                    │  └──────────┘ └───────┘ └───────┘         │
                    └─────────────────────────────────────────────┘
```

**Internal communication:** Services communicate via Docker DNS names (e.g., `http://ifcopenshell-worker:8000`). No ports need to be exposed for inter-service traffic.

**External exposure:** Only n8n (workflow UI + webhook endpoints), QGIS Server (WMS/WFS), and optionally web-ifc (if used as a standalone API) need published ports.

---

## 5. Integration Patterns Across All Four Domains

### 5.1 Pattern: File-Based Integration (IFC as Interchange)

The IFC file serves as the universal interchange format across all four domains:

1. **FreeCAD** authors/modifies the IFC file (NativeIFC mode)
2. **IfcOpenShell** (in Docker) extracts quantities and properties
3. **ERPNext** receives structured cost data via REST API
4. **n8n** orchestrates the pipeline and handles error routing

This is the simplest pattern but has limitations: file transfers are slow for large models, and there is no real-time synchronization.

### 5.2 Pattern: API-Based Integration (REST/GraphQL)

Each service exposes APIs that n8n connects:

| Service | API Type | Key Endpoints |
|---------|----------|---------------|
| ERPNext | REST | `/api/resource/BOM`, `/api/resource/Item` |
| Speckle | GraphQL | `/graphql` (streams, commits, objects) |
| IfcPipeline | REST | `/ifcconvert`, `/calculate-qtos`, `/ifcclash` |
| QGIS Server | OGC | WMS GetMap, WFS GetFeature |

### 5.3 Pattern: Event-Driven (Webhooks + Message Queues)

- **Speckle webhooks** trigger n8n workflows when models change
- **ERPNext webhooks** notify when BOMs are approved or costs change
- **Redis queues** (within Docker stack) handle async processing jobs
- **n8n sub-workflows** enable modular, reusable pipeline stages

### 5.4 End-to-End Pipeline Example

**Scenario:** Architect updates a building model in FreeCAD, and the costing in ERPNext must update automatically.

```
Step 1: Architect saves IFC file (FreeCAD NativeIFC, locked mode)
   │
Step 2: File watcher or Git webhook detects change
   │
Step 3: n8n Webhook node receives event
   │
Step 4: n8n Execute Command → IfcOpenShell validates IFC
   │    (Python script in ifcopenshell-worker container)
   │
Step 5: n8n Execute Command → extract quantities
   │    ifcopenshell.util.element.get_psets(element, qtos_only=True)
   │
Step 6: n8n Code node → transform IFC quantities to ERPNext BOM format
   │    Map IfcClassificationReference → Item Group
   │    Map Qto quantities → BOM Item quantities
   │    Convert IFC units → ERPNext UOM
   │
Step 7: n8n HTTP Request → POST /api/resource/Item (create missing items)
   │
Step 8: n8n HTTP Request → POST /api/resource/BOM (create/update BOM)
   │
Step 9: n8n HTTP Request → POST /api/resource/BOM/{name} (submit BOM)
   │
Step 10: n8n Send Email → notify cost manager of updated BOM
```

**Error handling at each step:**
- Step 4 failure: invalid IFC → notify architect, stop pipeline
- Step 7 failure: duplicate item → skip creation, use existing
- Step 8 failure: BOM validation error → log to ERPNext, notify manager
- Any step: n8n Error Trigger node catches unhandled exceptions

### 5.5 Resource Planning for Full Stack

| Component | CPU | RAM | Disk | Notes |
|-----------|-----|-----|------|-------|
| PostgreSQL | 1 core | 1GB | 10GB | Shared by Speckle + IfcPipeline |
| Redis | 0.5 core | 256MB | — | Job queues, caching |
| MinIO | 0.5 core | 512MB | 50GB+ | IFC file storage |
| IfcOpenShell worker | 2 cores | 4GB | 1GB | Memory scales with IFC file size |
| web-ifc worker | 1 core | 2GB | 500MB | WASM-based, fast geometry |
| QGIS Server | 1 core | 1GB | 5GB | Per concurrent WMS request |
| n8n | 1 core | 1GB | 1GB | Workflow orchestration |
| ERPNext (full) | 2 cores | 4GB | 20GB | Separate deployment recommended |
| **Total (without ERPNext)** | **~7 cores** | **~10GB** | **~68GB** | Minimum for development |

For production deployments processing models larger than 500MB, double the IfcOpenShell worker memory allocation and consider running multiple worker replicas behind a load balancer.

---

## 6. Version Reference

| Technology | Version | Notes |
|------------|---------|-------|
| IfcOpenShell | 0.8.x | conda-forge, Python 3.10+ |
| FreeCAD | 1.0+ | NativeIFC requires 1.0 minimum |
| n8n | 1.x | Self-hosted, docker image `n8nio/n8n` |
| ERPNext | v15 | Frappe Framework v15 |
| Speckle Server | 2.x | Six-service Docker deployment |
| QGIS Server | 3.34 LTR | Official `qgis/qgis-server:ltr` image |
| web-ifc | 0.0.57 | npm package, Node.js 20+ |
| Docker Compose | v2 | `docker compose` (not `docker-compose`) |
| PostgreSQL | 16 | Alpine variant for smaller image |
| Redis | 7 | Alpine variant |
| MinIO | latest | S3-compatible object storage |

---

## 7. Sources

- IfcOpenShell documentation: https://docs.ifcopenshell.org/
- IfcOpenShell code examples: https://docs.ifcopenshell.org/ifcopenshell-python/code_examples.html
- IfcOpenShell pset API: https://docs.ifcopenshell.org/autoapi/ifcopenshell/api/pset/index.html
- FreeCAD NativeIFC: https://wiki.freecad.org/NativeIFC
- FreeCAD BIM Workbench: https://wiki.freecad.org/BIM_Workbench
- ERPNext BOM documentation: https://docs.frappe.io/erpnext/user/manual/en/bill-of-materials
- Frappe REST API: https://docs.frappe.io/framework/user/en/api/rest
- n8n core nodes: https://docs.n8n.io/integrations/builtin/core-nodes/
- n8n Docker deployment: https://docs.n8n.io/hosting/installation/server-setups/docker-compose/
- Speckle Server Docker: https://github.com/specklesystems/speckle-server
- Speckle docker-compose: https://github.com/specklesystems/speckle-server/blob/main/docker-compose-speckle.yml
- QGIS Server Docker: https://github.com/qgis/qgis-docker
- web-ifc npm: https://www.npmjs.com/package/web-ifc
- web-ifc GitHub: https://github.com/ThatOpen/engine_web-ifc
- IfcPipeline: https://github.com/jonatanjacobsson/ifcpipeline
- n8n-nodes-ifcpipeline: https://github.com/jonatanjacobsson/n8n-nodes-ifcpipeline
- Docker Hub — IfcOpenShell images: https://hub.docker.com/r/aecgeeks/ifcopenshell
- Frappe Forum — BOM REST API: https://discuss.frappe.io/t/create-bom-using-rest-api/21593
