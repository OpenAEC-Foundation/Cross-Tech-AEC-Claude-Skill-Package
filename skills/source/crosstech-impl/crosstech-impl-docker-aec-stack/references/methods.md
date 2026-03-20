# crosstech-impl-docker-aec-stack -- Methods Reference

## Dockerfile Patterns

### IfcOpenShell Worker (Conda-Based, Full Geometry)

```dockerfile
FROM continuumio/miniconda3:24.7.1-0

# ALWAYS use conda-forge for IfcOpenShell + OCCT geometry bindings
RUN conda install -c conda-forge ifcopenshell pythonocc-core -y && \
    conda clean -afy

# Application layer
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

**When to use:** Any service that processes IFC geometry (clash detection, quantity extraction with geometry, 3D visualization data).

**Image size:** ~1GB (400MB base + 600MB IfcOpenShell/OCCT).

### IfcOpenShell Worker (Pip-Based, Properties Only)

```dockerfile
FROM python:3.11-slim

RUN pip install --no-cache-dir \
    ifcopenshell==0.8.0 \
    fastapi==0.115.0 \
    uvicorn==0.30.0

WORKDIR /app
COPY ./scripts /app/scripts

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**When to use:** Services that ONLY extract properties, classifications, and spatial structure -- no geometry processing.

**Image size:** ~300MB. NEVER use this variant if geometry operations are needed -- they will fail silently or raise ImportError.

### web-ifc Worker (Node.js + WASM)

```dockerfile
FROM node:20-slim

WORKDIR /app

RUN npm init -y && \
    npm install web-ifc@0.0.57

COPY ./src /app/src

EXPOSE 3000
CMD ["node", "src/server.js"]
```

**When to use:** Fast geometry extraction for Three.js viewers, lightweight IFC parsing, browser-compatible output.

**Image size:** ~200MB.

### QGIS Server

QGIS Server uses the official pre-built image. No custom Dockerfile needed.

```yaml
image: qgis/qgis-server:ltr
```

**Alternative:** `kartoza/qgis-server` provides additional configuration options (custom fonts, plugins, PostgreSQL/PostGIS integration).

---

## docker-compose Service Definitions

### Infrastructure Services

```yaml
postgres:
  image: postgres:16-alpine
  environment:
    POSTGRES_USER: aec
    POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"  # ALWAYS use env var
    POSTGRES_DB: aec
  volumes:
    - postgres_data:/var/lib/postgresql/data
  ports:
    - "5432:5432"  # Remove in production
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U aec"]
    interval: 10s
    timeout: 5s
    retries: 5

redis:
  image: redis:7-alpine
  ports:
    - "6379:6379"  # Remove in production
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5

minio:
  image: minio/minio:latest
  command: server /data --console-address ":9001"
  environment:
    MINIO_ROOT_USER: "${MINIO_ROOT_USER}"
    MINIO_ROOT_PASSWORD: "${MINIO_ROOT_PASSWORD}"
  volumes:
    - minio_data:/data
  ports:
    - "9000:9000"
    - "9001:9001"  # Console UI
  healthcheck:
    test: ["CMD", "mc", "ready", "local"]
    interval: 10s
    timeout: 5s
    retries: 5
```

### AEC Processing Services

```yaml
ifcopenshell-worker:
  build:
    context: ./services/ifcopenshell
    dockerfile: Dockerfile
  volumes:
    - ifc_data:/data/ifc
    - ./scripts:/app/scripts:ro
  environment:
    REDIS_URL: redis://redis:6379
    POSTGRES_URL: postgresql://aec:${POSTGRES_PASSWORD}@postgres/aec
  depends_on:
    redis:
      condition: service_healthy
    postgres:
      condition: service_healthy
  deploy:
    resources:
      limits:
        memory: 4G
  restart: always

web-ifc-worker:
  build:
    context: ./services/web-ifc
    dockerfile: Dockerfile
  volumes:
    - ifc_data:/data/ifc:ro  # Read-only -- web-ifc only reads IFC files
  ports:
    - "3001:3000"
  deploy:
    resources:
      limits:
        memory: 2G
  restart: always

qgis-server:
  image: qgis/qgis-server:ltr
  volumes:
    - ./qgis-projects:/project:ro
    - ./gis-data:/data:ro
  ports:
    - "8010:80"
  environment:
    QGIS_PROJECT_FILE: /project/project.qgs
  restart: always
```

### Workflow Orchestration

```yaml
n8n:
  image: n8nio/n8n:latest
  ports:
    - "5678:5678"
  environment:
    N8N_BASIC_AUTH_ACTIVE: "true"
    N8N_BASIC_AUTH_USER: "${N8N_USER}"
    N8N_BASIC_AUTH_PASSWORD: "${N8N_PASSWORD}"
    EXECUTIONS_MODE: queue
    QUEUE_BULL_REDIS_HOST: redis
    GENERIC_TIMEZONE: Europe/Amsterdam
  volumes:
    - n8n_data:/home/node/.n8n
    - ifc_data:/data/ifc
  depends_on:
    - redis
  restart: always
```

---

## Volume Strategies

### Named Volumes (Single-Node)

```yaml
volumes:
  postgres_data:   # Database persistence
  minio_data:      # Object storage persistence
  n8n_data:        # Workflow definitions + credentials
  ifc_data:        # Shared IFC file storage
```

**Directory structure inside `ifc_data`:**

```
/data/ifc/
  {project_id}/
    {filename}.ifc          # Original IFC files
    {filename}.quantities.json  # Extracted quantities
    {filename}.classes.json     # Classification data
  tmp/                       # Ephemeral processing artifacts
```

### Object Storage (Multi-Node)

For deployments spanning multiple hosts, use MinIO/S3 instead of named volumes:

```python
# Upload IFC file to MinIO (Python)
import boto3

s3 = boto3.client('s3',
    endpoint_url='http://minio:9000',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadmin')

s3.upload_file('model.ifc', 'ifc-files', 'project-001/model.ifc')
```

### Tmpfs for Ephemeral Processing

```yaml
ifcopenshell-worker:
  tmpfs:
    - /tmp/ifc-processing:size=2G
```

Use tmpfs for intermediate files (converted formats, extracted meshes) that do not need persistence.

---

## Speckle Server Deployment

Speckle Server requires two compose files. ALWAYS deploy infrastructure first.

### Infrastructure (docker-compose-deps.yml)

| Service | Image | Purpose |
|---------|-------|---------|
| PostgreSQL | `postgres:14` | Primary database |
| Redis | `redis:7-alpine` | Caching, job queues |
| MinIO | `minio/minio` | S3-compatible blob storage |

### Application (docker-compose-speckle.yml)

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| `speckle-server` | `speckle/speckle-server` | 3000 | Core API (GraphQL + REST) |
| `speckle-frontend-2` | `speckle/speckle-frontend-2` | -- | Nuxt.js web frontend |
| `speckle-ingress` | `speckle/speckle-docker-compose-ingress` | 80 | Nginx reverse proxy |
| `preview-service` | `speckle/speckle-preview-service` | 3001 | 3D preview generation |
| `webhook-service` | `speckle/speckle-webhook-service` | -- | Outbound webhook delivery |
| `ifc-import-server` | `speckle/ifc-import-service` | -- | IFC file import processing |

### Critical Environment Variables

```yaml
CANONICAL_URL: "https://speckle.example.com"       # ALWAYS set to actual domain
SESSION_SECRET: "${SPECKLE_SESSION_SECRET}"          # NEVER use default
STRATEGY_LOCAL: "true"
POSTGRES_URL: "postgres://speckle:${SPECKLE_DB_PASS}@postgres/speckle"
REDIS_URL: "redis://redis"
S3_ENDPOINT: "http://minio:9000"
S3_ACCESS_KEY: "${MINIO_ACCESS_KEY}"
S3_SECRET_KEY: "${MINIO_SECRET_KEY}"
FILE_SIZE_LIMIT_MB: 1000
```

---

## Healthcheck Patterns

ALWAYS add healthchecks to production services:

```yaml
# PostgreSQL
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U aec"]
  interval: 10s
  timeout: 5s
  retries: 5

# Redis
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 10s
  timeout: 5s
  retries: 5

# HTTP-based services (n8n, IfcOpenShell worker)
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 30s
```

Use `depends_on` with `condition: service_healthy` to enforce startup order.

---

## Network Configuration

```yaml
networks:
  aec-net:
    driver: bridge

services:
  ifcopenshell-worker:
    networks:
      - aec-net
  web-ifc-worker:
    networks:
      - aec-net
  # ... all services on aec-net
```

Services resolve each other by container name: `http://ifcopenshell-worker:8000`, `http://redis:6379`.

NEVER hard-code IP addresses -- Docker DNS handles resolution automatically.
