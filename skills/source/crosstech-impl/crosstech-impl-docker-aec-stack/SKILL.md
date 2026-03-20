---
name: crosstech-impl-docker-aec-stack
description: >
  Use when containerizing AEC tools or deploying multi-service AEC stacks with Docker.
  Prevents the common mistake of missing geometry dependencies (OCCT/OCC) in
  IfcOpenShell containers or misconfiguring Speckle Server services.
  Covers Dockerfiles for IfcOpenShell, Speckle Server deployment, QGIS Server,
  web-ifc workers, and multi-service docker-compose orchestration.
  Keywords: Docker, AEC, IfcOpenShell container, Speckle Server, QGIS Server,
  docker-compose, web-ifc, containerization, AEC deployment.
license: MIT
compatibility: "Designed for Claude Code. Requires Docker Engine 24+ and Docker Compose v2."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# crosstech-impl-docker-aec-stack

## Quick Reference

| AEC Service | Base Image | Exposed Port | Memory Limit | Disk |
|-------------|-----------|-------------|-------------|------|
| IfcOpenShell worker | `continuumio/miniconda3:24.7.1-0` | 8000 (internal) | 4GB | 1GB |
| web-ifc worker | `node:20-slim` | 3000 (internal) | 2GB | 500MB |
| QGIS Server | `qgis/qgis-server:ltr` | 80 | 1GB | 5GB |
| Speckle Server | `speckle/speckle-server` | 3000 | 4GB total (6 services) | 20GB |
| n8n | `n8nio/n8n:latest` | 5678 | 1GB | 1GB |
| PostgreSQL | `postgres:16-alpine` | 5432 | 1GB | 10GB |
| Redis | `redis:7-alpine` | 6379 | 256MB | -- |
| MinIO | `minio/minio:latest` | 9000/9001 | 512MB | 50GB+ |

**Full stack minimum (without ERPNext):** 7 cores, 10GB RAM, 68GB disk.

### Critical Warnings

**ALWAYS** use `docker compose` (v2 plugin syntax), **NEVER** `docker-compose` (legacy standalone binary).

**ALWAYS** install IfcOpenShell via conda (not pip) when geometry processing is required -- pip installs lack OCCT bindings.

**NEVER** use default secrets in production -- ALWAYS generate unique values for `SESSION_SECRET`, database passwords, and MinIO credentials.

**NEVER** expose infrastructure ports (PostgreSQL, Redis, MinIO) to the host in production -- only AEC service ports need external access.

**ALWAYS** set memory limits on IfcOpenShell workers -- without limits, a large IFC file (>500MB) will consume all host memory.

---

## Technology Boundary

### Side A: Docker (Containers, Compose)

| Aspect | Detail |
|--------|--------|
| Technology | Docker Engine 24+, Docker Compose v2 |
| Data format | Dockerfiles, docker-compose.yml (YAML) |
| API surface | `docker compose up/down/build`, Dockerfile instructions |
| Networking | Docker bridge networks, DNS-based service discovery |
| Storage | Named volumes, bind mounts, tmpfs |

### Side B: AEC Services (IfcOpenShell, Speckle, QGIS, web-ifc)

| Aspect | Detail |
|--------|--------|
| IfcOpenShell | Python 3.10+, requires OCCT via conda-forge, IFC4/IFC4X3 |
| Speckle Server | 6 application services + 3 infrastructure services |
| QGIS Server | OGC-compliant WMS/WFS/WCS, QGIS 3.34 LTR |
| web-ifc | Node.js 20+, WebAssembly-based IFC parser, v0.0.57 |
| n8n | Workflow automation, webhook + HTTP nodes |

### The Bridge: Dockerfiles + docker-compose Orchestration

Docker solves three AEC deployment problems:

1. **Dependency hell** -- IfcOpenShell requires OCCT, which requires specific C++ libraries. Conda inside a container isolates this completely.
2. **Service coordination** -- Speckle Server alone requires 9 coordinated services. Docker Compose defines them declaratively.
3. **File sharing** -- IFC files (50MB-2GB) must be accessible to multiple processing services. Named volumes provide a shared filesystem.

**Data flow:** IFC files enter via n8n webhooks or direct upload to MinIO. Processing services (IfcOpenShell, web-ifc) read from the shared `ifc_data` volume. Results flow to PostgreSQL (structured data) or back through the n8n pipeline to ERPNext/Speckle APIs.

---

## Critical Rules

1. **ALWAYS** use conda-forge for IfcOpenShell in Docker -- `conda install -c conda-forge ifcopenshell pythonocc-core`. The `pythonocc-core` package provides OCCT geometry bindings.
2. **ALWAYS** clean conda cache after install: `conda clean -afy`. This reduces image size by 200-400MB.
3. **ALWAYS** set `deploy.resources.limits.memory` on IfcOpenShell and web-ifc workers. IFC geometry processing has unbounded memory growth on large models.
4. **ALWAYS** use `platform: linux/amd64` for Speckle Server images -- they are NOT available for ARM64.
5. **ALWAYS** use `restart: always` for production AEC services.
6. **NEVER** mount IFC data directories with `:rw` on services that only read -- use `:ro` (read-only) to prevent accidental file corruption.
7. **ALWAYS** use sub-directories per project in the IFC volume: `/data/ifc/{project_id}/{filename}.ifc`.
8. **NEVER** run ERPNext in the same compose stack as the AEC processing services -- it requires its own MariaDB, Redis, and bench-worker infrastructure. Connect via REST API from n8n.

---

## Decision Tree

```
Need to containerize an AEC tool?
├── Is it IfcOpenShell (Python)?
│   ├── Needs geometry processing? → conda-based Dockerfile (miniconda3 + pythonocc-core)
│   └── Property extraction only? → pip-based Dockerfile (python:3.11-slim + ifcopenshell)
├── Is it web-ifc (JavaScript)?
│   └── Use node:20-slim + npm install web-ifc
├── Is it QGIS Server?
│   └── Use qgis/qgis-server:ltr (or kartoza/qgis-server for extra config)
├── Is it Speckle Server?
│   └── Use official speckle docker-compose (9 services total)
└── Is it a multi-service AEC stack?
    └── Compose all services with shared ifc_data volume and aec-net network

Volume strategy for IFC files?
├── Single-node deployment → Named volume (ifc_data)
├── Multi-node deployment → MinIO/S3 object storage
└── Temporary processing artifacts → tmpfs mount

Which ports to expose externally?
├── n8n: 5678 (workflow UI + webhooks) → ALWAYS expose
├── QGIS Server: 8010→80 (WMS/WFS) → Expose if GIS clients need access
├── web-ifc: 3001→3000 → Expose ONLY if used as standalone API
└── Infrastructure (postgres, redis, minio) → NEVER expose in production
```

---

## Essential Patterns

### Pattern 1: IfcOpenShell Dockerfile (Conda-Based with OCCT)

```dockerfile
FROM continuumio/miniconda3:24.7.1-0

# ALWAYS install both ifcopenshell AND pythonocc-core for geometry support
RUN conda install -c conda-forge ifcopenshell pythonocc-core -y && \
    conda clean -afy

# Application dependencies
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

**Image size:** ~1GB (400MB miniconda3 + 600MB IfcOpenShell/OCCT).

### Pattern 2: web-ifc Worker Dockerfile

```dockerfile
FROM node:20-slim

WORKDIR /app

RUN npm init -y && \
    npm install web-ifc@0.0.57

COPY ./src /app/src

EXPOSE 3000
CMD ["node", "src/server.js"]
```

**Image size:** ~200MB (node:20-slim + WASM binaries).

### Pattern 3: QGIS Server Service

```yaml
qgis-server:
  image: qgis/qgis-server:ltr
  ports:
    - "8010:80"
  volumes:
    - ./qgis-projects:/project:ro
    - ./gis-data:/data:ro
  environment:
    QGIS_PROJECT_FILE: /project/project.qgs
```

### Pattern 4: Shared IFC Volume

```yaml
volumes:
  ifc_data:    # Named volume shared across AEC services

services:
  ifcopenshell-worker:
    volumes:
      - ifc_data:/data/ifc       # Read-write for processing output
  web-ifc-worker:
    volumes:
      - ifc_data:/data/ifc:ro    # Read-only -- web-ifc only reads
  n8n:
    volumes:
      - ifc_data:/data/ifc       # Read-write for file ingestion
```

---

## Common Operations

### Start the Full AEC Stack

```bash
# Start infrastructure first, then AEC services
docker compose up -d postgres redis minio
docker compose up -d ifcopenshell-worker web-ifc-worker qgis-server n8n
```

### Verify Service Health

```bash
# Check all services are running
docker compose ps

# Test IfcOpenShell worker
curl http://localhost:8000/health

# Test QGIS Server (WMS GetCapabilities)
curl "http://localhost:8010/ogc/project?SERVICE=WMS&REQUEST=GetCapabilities"

# Test n8n
curl http://localhost:5678/healthz
```

### Scale IfcOpenShell Workers for Large Models

```bash
# Run 3 parallel workers for batch IFC processing
docker compose up -d --scale ifcopenshell-worker=3
```

### Process an IFC File Through the Stack

```bash
# Copy IFC file into the shared volume
docker compose cp ./model.ifc ifcopenshell-worker:/data/ifc/project-001/model.ifc

# Trigger processing via n8n webhook (example)
curl -X POST http://localhost:5678/webhook/ifc-process \
  -H "Content-Type: application/json" \
  -d '{"project_id": "project-001", "filename": "model.ifc"}'
```

### Speckle Server Deployment (Separate Stack)

```bash
# Clone Speckle Server repository
git clone https://github.com/specklesystems/speckle-server.git
cd speckle-server

# Start infrastructure
docker compose -f docker-compose-deps.yml up -d

# Configure environment variables (ALWAYS change defaults)
cp .env.example .env
# Edit .env: set CANONICAL_URL, SESSION_SECRET, S3 keys

# Start application services
docker compose -f docker-compose-speckle.yml up -d
```

### Network Architecture

```
                    +---------------------------------------------+
                    |              Docker Network (aec-net)        |
                    |                                              |
  External          |  +----------+    +--------------------+      |
  API calls  ------>|  |   n8n    |--->| ifcopenshell-worker|      |
  (port 5678)       |  |  :5678   |    | :8000 (internal)   |      |
                    |  +----+-----+    +--------------------+      |
                    |       |                                       |
                    |       +--------->+------------------+        |
                    |       |          |  web-ifc-worker  |        |
                    |       |          |  :3000 (internal)|        |
                    |       |          +------------------+        |
                    |       |                                       |
  WMS/WFS   ------>|  +----v-----+    +------------------+        |
  (port 8010)       |  |  qgis   |    |    ERPNext       |        |
                    |  |  server  |    |  (external/API)  |        |
                    |  +----------+    +------------------+        |
                    |                                              |
                    |  +----------+ +-------+ +-------+            |
                    |  | postgres | | redis | | minio |            |
                    |  +----------+ +-------+ +-------+            |
                    +---------------------------------------------+
```

Services communicate via Docker DNS names (e.g., `http://ifcopenshell-worker:8000`). No ports need exposure for inter-service traffic.

---

## Resource Planning

| Component | CPU | RAM | Disk | Scaling Notes |
|-----------|-----|-----|------|---------------|
| PostgreSQL | 1 core | 1GB | 10GB | Shared by Speckle + pipeline data |
| Redis | 0.5 core | 256MB | -- | Job queues, caching |
| MinIO | 0.5 core | 512MB | 50GB+ | IFC file storage, scales with project count |
| IfcOpenShell worker | 2 cores | 4GB | 1GB | Memory scales linearly with IFC file size |
| web-ifc worker | 1 core | 2GB | 500MB | WASM-based, faster geometry than IfcOpenShell |
| QGIS Server | 1 core | 1GB | 5GB | Memory per concurrent WMS request |
| n8n | 1 core | 1GB | 1GB | Workflow orchestration hub |
| **Total (dev)** | **~7 cores** | **~10GB** | **~68GB** | Minimum for development |

For IFC files >500MB: double IfcOpenShell worker memory. For concurrent users: scale workers with `--scale`.

---

## Reference Links

- [references/methods.md](references/methods.md) -- Dockerfile patterns, docker-compose service definitions, volume strategies
- [references/examples.md](references/examples.md) -- Complete docker-compose.yml for full AEC development stack
- [references/anti-patterns.md](references/anti-patterns.md) -- Docker AEC deployment mistakes and how to avoid them

### Official Sources

- Docker Compose specification: https://docs.docker.com/compose/compose-file/
- Speckle Server Docker: https://github.com/specklesystems/speckle-server
- QGIS Server Docker: https://github.com/qgis/qgis-docker
- web-ifc npm: https://www.npmjs.com/package/web-ifc
- IfcOpenShell conda-forge: https://anaconda.org/conda-forge/ifcopenshell
- n8n Docker deployment: https://docs.n8n.io/hosting/installation/server-setups/docker-compose/
- Docker Hub -- IfcOpenShell images: https://hub.docker.com/r/aecgeeks/ifcopenshell
