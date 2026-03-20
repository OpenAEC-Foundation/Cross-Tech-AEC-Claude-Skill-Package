# crosstech-impl-docker-aec-stack -- Examples Reference

## Example 1: Complete AEC Development Stack (docker-compose.yml)

This is a production-ready docker-compose file for an AEC development environment with all core services.

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
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD:-aec_dev_password}"
      POSTGRES_DB: aec
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U aec"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: "${MINIO_ROOT_USER:-minioadmin}"
      MINIO_ROOT_PASSWORD: "${MINIO_ROOT_PASSWORD:-minioadmin}"
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
      POSTGRES_URL: postgresql://aec:${POSTGRES_PASSWORD:-aec_dev_password}@postgres/aec
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 4G
    restart: unless-stopped

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
    restart: unless-stopped

  qgis-server:
    image: qgis/qgis-server:ltr
    volumes:
      - ./qgis-projects:/project:ro
      - ./gis-data:/data:ro
    ports:
      - "8010:80"
    environment:
      QGIS_PROJECT_FILE: /project/project.qgs
    restart: unless-stopped

  # ============================================
  # Workflow Automation
  # ============================================
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      N8N_BASIC_AUTH_ACTIVE: "true"
      N8N_BASIC_AUTH_USER: "${N8N_USER:-admin}"
      N8N_BASIC_AUTH_PASSWORD: "${N8N_PASSWORD:-change_me}"
      EXECUTIONS_MODE: queue
      QUEUE_BULL_REDIS_HOST: redis
      GENERIC_TIMEZONE: Europe/Amsterdam
    volumes:
      - n8n_data:/home/node/.n8n
      - ifc_data:/data/ifc
    depends_on:
      - redis
    restart: unless-stopped

  # ============================================
  # Business Logic (Optional Profile)
  # ============================================
  erpnext:
    # ERPNext requires its own compose stack -- reference only
    # Full deployment: https://github.com/frappe/frappe_docker
    image: frappe/erpnext:v15
    profiles:
      - erpnext  # Only start with: docker compose --profile erpnext up

volumes:
  postgres_data:
  minio_data:
  n8n_data:
  ifc_data:

networks:
  default:
    name: aec-net
```

### Usage

```bash
# Start the full stack
docker compose up -d

# Start with ERPNext (optional)
docker compose --profile erpnext up -d

# View logs for a specific service
docker compose logs -f ifcopenshell-worker

# Stop everything
docker compose down

# Stop and remove volumes (DESTROYS ALL DATA)
docker compose down -v
```

---

## Example 2: IfcOpenShell Worker with FastAPI

### services/ifcopenshell/Dockerfile

```dockerfile
FROM continuumio/miniconda3:24.7.1-0

RUN conda install -c conda-forge ifcopenshell pythonocc-core -y && \
    conda clean -afy

RUN pip install --no-cache-dir \
    fastapi==0.115.0 \
    uvicorn==0.30.0 \
    redis==5.0.0 \
    rq==1.16.0 \
    python-multipart==0.0.9

WORKDIR /app
COPY . /app

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### services/ifcopenshell/main.py

```python
import os
import json
from pathlib import Path
from fastapi import FastAPI, UploadFile
import ifcopenshell
import ifcopenshell.util.element

app = FastAPI(title="IfcOpenShell Worker")
IFC_DIR = Path("/data/ifc")


@app.get("/health")
def health():
    return {"status": "ok", "ifcopenshell_version": ifcopenshell.version}


@app.post("/extract-quantities/{project_id}")
def extract_quantities(project_id: str, filename: str):
    """Extract all quantity sets from an IFC file."""
    ifc_path = IFC_DIR / project_id / filename
    if not ifc_path.exists():
        return {"error": f"File not found: {ifc_path}"}

    model = ifcopenshell.open(str(ifc_path))
    results = []

    for element in model.by_type("IfcBuildingElement"):
        qtos = ifcopenshell.util.element.get_psets(element, qtos_only=True)
        if qtos:
            results.append({
                "global_id": element.GlobalId,
                "ifc_class": element.is_a(),
                "name": element.Name,
                "quantities": qtos,
            })

    # Write results to JSON alongside the IFC file
    output_path = ifc_path.with_suffix(".quantities.json")
    with open(output_path, "w") as f:
        json.dump(results, f, indent=2)

    return {"count": len(results), "output": str(output_path)}


@app.post("/upload/{project_id}")
async def upload_ifc(project_id: str, file: UploadFile):
    """Upload an IFC file to the shared volume."""
    project_dir = IFC_DIR / project_id
    project_dir.mkdir(parents=True, exist_ok=True)

    dest = project_dir / file.filename
    with open(dest, "wb") as f:
        content = await file.read()
        f.write(content)

    return {"path": str(dest), "size_mb": round(len(content) / 1_000_000, 2)}
```

---

## Example 3: web-ifc Worker with Express

### services/web-ifc/Dockerfile

```dockerfile
FROM node:20-slim

WORKDIR /app

RUN npm init -y && \
    npm install web-ifc@0.0.57 express@4.18.2

COPY ./src /app/src

EXPOSE 3000
CMD ["node", "src/server.js"]
```

### services/web-ifc/src/server.js

```javascript
const express = require("express");
const fs = require("fs");
const path = require("path");
const WebIFC = require("web-ifc/web-ifc-api-node.js");

const app = express();
const IFC_DIR = "/data/ifc";

app.get("/health", (req, res) => {
  res.json({ status: "ok", engine: "web-ifc" });
});

app.get("/geometry/:projectId/:filename", async (req, res) => {
  const filePath = path.join(
    IFC_DIR,
    req.params.projectId,
    req.params.filename
  );

  if (!fs.existsSync(filePath)) {
    return res.status(404).json({ error: "File not found" });
  }

  const ifcApi = new WebIFC.IfcAPI();
  await ifcApi.Init();

  const data = fs.readFileSync(filePath);
  const modelID = ifcApi.OpenModel(data);

  // Extract wall geometry for Three.js
  const walls = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCWALL);
  const geometries = [];

  for (let i = 0; i < walls.size(); i++) {
    const wallId = walls.get(i);
    const props = ifcApi.GetLine(modelID, wallId);
    geometries.push({
      expressId: wallId,
      globalId: props.GlobalId?.value,
      name: props.Name?.value,
    });
  }

  ifcApi.CloseModel(modelID);

  res.json({
    wallCount: walls.size(),
    walls: geometries,
  });
});

app.listen(3000, () => {
  console.log("web-ifc worker listening on port 3000");
});
```

---

## Example 4: Environment File (.env)

ALWAYS use a `.env` file for secrets. NEVER commit this file to Git.

```env
# Infrastructure
POSTGRES_PASSWORD=secure_random_password_here
MINIO_ROOT_USER=aec_minio_admin
MINIO_ROOT_PASSWORD=secure_random_password_here

# n8n
N8N_USER=admin
N8N_PASSWORD=secure_random_password_here

# Speckle (if deploying)
SPECKLE_SESSION_SECRET=generate_64_char_random_string
SPECKLE_DB_PASS=secure_random_password_here
MINIO_ACCESS_KEY=speckle_minio_key
MINIO_SECRET_KEY=secure_random_password_here
```

```bash
# Generate random secrets
openssl rand -hex 32
```

---

## Example 5: Production Hardening

### Remove Development Ports

```yaml
# Production: remove host port bindings for infrastructure
postgres:
  image: postgres:16-alpine
  # NO ports: section -- only accessible within Docker network

redis:
  image: redis:7-alpine
  # NO ports: section

minio:
  image: minio/minio:latest
  ports:
    - "9001:9001"  # Console only, behind auth
  # Remove 9000 -- only internal S3 access needed
```

### Add Resource Limits to All Services

```yaml
deploy:
  resources:
    limits:
      cpus: "2.0"
      memory: 4G
    reservations:
      cpus: "0.5"
      memory: 1G
```

### Enable Restart Policies

```yaml
restart: always  # Production
restart: unless-stopped  # Development
```

### Log Rotation

```yaml
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
```

---

## Example 6: Scaling IfcOpenShell Workers

For batch processing many IFC files, scale workers horizontally:

```bash
# Run 3 parallel IfcOpenShell workers
docker compose up -d --scale ifcopenshell-worker=3
```

Workers pull jobs from a Redis queue (RQ pattern):

```python
# worker.py -- run inside each ifcopenshell-worker container
from redis import Redis
from rq import Worker, Queue

redis_conn = Redis.from_url("redis://redis:6379")
queue = Queue("ifc-processing", connection=redis_conn)

worker = Worker([queue], connection=redis_conn)
worker.work()
```

```python
# enqueue.py -- called by n8n or API endpoint
from redis import Redis
from rq import Queue

redis_conn = Redis.from_url("redis://redis:6379")
queue = Queue("ifc-processing", connection=redis_conn)

job = queue.enqueue(
    "tasks.extract_quantities",
    "/data/ifc/project-001/model.ifc",
    job_timeout=600  # 10 minutes for large files
)
```
