# crosstech-impl-docker-aec-stack -- Anti-Patterns Reference

## Anti-Pattern 1: Installing IfcOpenShell via pip for Geometry Processing

### What Goes Wrong

```dockerfile
# WRONG -- pip install lacks OCCT geometry bindings
FROM python:3.11-slim
RUN pip install ifcopenshell
```

IfcOpenShell installed via pip does NOT include OpenCASCADE Technology (OCCT) geometry bindings. Any call to geometry-related functions (`ifcopenshell.geom.create_shape()`, geometry validation, clash detection) will either raise `ImportError` or silently return empty results.

### Why It Happens

Developers assume pip installs the complete package. The pip distribution of IfcOpenShell is a minimal build -- it supports IFC schema reading and property extraction but NOT geometry processing.

### Correct Approach

```dockerfile
# CORRECT -- conda-forge includes OCCT geometry bindings
FROM continuumio/miniconda3:24.7.1-0
RUN conda install -c conda-forge ifcopenshell pythonocc-core -y && \
    conda clean -afy
```

ALWAYS use conda-forge when geometry processing is required. The `pythonocc-core` package provides the OCCT bindings that IfcOpenShell needs for geometry operations.

---

## Anti-Pattern 2: Running All Services Without Memory Limits

### What Goes Wrong

```yaml
# WRONG -- no memory limits
ifcopenshell-worker:
  build: ./services/ifcopenshell
  volumes:
    - ifc_data:/data/ifc
```

A single large IFC file (>500MB) causes the IfcOpenShell worker to consume 8-16GB of memory, triggering the Linux OOM killer and crashing the entire Docker host -- taking down all other AEC services.

### Why It Happens

IFC geometry processing loads the entire model into memory. Memory usage scales linearly (and sometimes superlinearly) with file size. Without limits, one runaway process affects all co-located services.

### Correct Approach

```yaml
# CORRECT -- enforce memory ceiling
ifcopenshell-worker:
  build: ./services/ifcopenshell
  volumes:
    - ifc_data:/data/ifc
  deploy:
    resources:
      limits:
        memory: 4G
```

ALWAYS set `deploy.resources.limits.memory` on IfcOpenShell and web-ifc workers. The container receives a controlled OOM kill instead of consuming all host memory.

---

## Anti-Pattern 3: Using Default Secrets in Production

### What Goes Wrong

```yaml
# WRONG -- hardcoded default passwords
postgres:
  environment:
    POSTGRES_PASSWORD: aec_dev_password

minio:
  environment:
    MINIO_ROOT_USER: minioadmin
    MINIO_ROOT_PASSWORD: minioadmin

n8n:
  environment:
    N8N_BASIC_AUTH_PASSWORD: change_me
```

Default credentials are published in documentation and example files. Any network-accessible service with default credentials is trivially compromised.

### Correct Approach

```yaml
# CORRECT -- use .env file with generated secrets
postgres:
  environment:
    POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"

minio:
  environment:
    MINIO_ROOT_USER: "${MINIO_ROOT_USER}"
    MINIO_ROOT_PASSWORD: "${MINIO_ROOT_PASSWORD}"
```

```bash
# Generate secrets
openssl rand -hex 32 > /dev/null  # Use output as password
```

ALWAYS use environment variable substitution from a `.env` file. NEVER commit the `.env` file to version control.

---

## Anti-Pattern 4: Exposing Infrastructure Ports to Host

### What Goes Wrong

```yaml
# WRONG -- infrastructure ports exposed externally
postgres:
  ports:
    - "5432:5432"  # PostgreSQL accessible from outside Docker

redis:
  ports:
    - "6379:6379"  # Redis accessible from outside Docker

minio:
  ports:
    - "9000:9000"  # MinIO S3 API accessible externally
```

Exposing database and cache ports makes them accessible to the entire network. Redis has no authentication by default. PostgreSQL and MinIO with weak passwords become attack vectors.

### Correct Approach

```yaml
# CORRECT -- no host port mapping for infrastructure
postgres:
  image: postgres:16-alpine
  # No ports: section -- only reachable via Docker network

redis:
  image: redis:7-alpine
  # No ports: section

minio:
  image: minio/minio:latest
  ports:
    - "9001:9001"  # Console UI only (authenticated)
  # Remove 9000 -- S3 API only needed internally
```

In production, ONLY expose ports for services that need external access: n8n (5678), QGIS Server (8010), and optionally web-ifc (3001). Infrastructure services communicate via the Docker bridge network.

---

## Anti-Pattern 5: Using `docker-compose` (v1) Instead of `docker compose` (v2)

### What Goes Wrong

```bash
# WRONG -- legacy standalone binary
docker-compose up -d
docker-compose down
```

`docker-compose` (with hyphen) is the legacy Python-based tool that is EOL and no longer maintained. It lacks features like `deploy.resources.limits`, `depends_on.condition`, and GPU support.

### Correct Approach

```bash
# CORRECT -- Docker Compose v2 plugin
docker compose up -d
docker compose down
```

ALWAYS use `docker compose` (space, no hyphen). This is the Go-based plugin integrated into Docker CLI. Verify with: `docker compose version`.

---

## Anti-Pattern 6: Mounting IFC Data as Read-Write on All Services

### What Goes Wrong

```yaml
# WRONG -- all services have write access
ifcopenshell-worker:
  volumes:
    - ifc_data:/data/ifc

web-ifc-worker:
  volumes:
    - ifc_data:/data/ifc      # Write access to shared data

qgis-server:
  volumes:
    - ifc_data:/data/ifc      # Write access to shared data
```

Multiple services writing to the same directory creates race conditions. A crashed web-ifc worker may leave partially written files that corrupt processing for other services.

### Correct Approach

```yaml
# CORRECT -- read-only for consumers, read-write for producers only
ifcopenshell-worker:
  volumes:
    - ifc_data:/data/ifc      # Read-write: produces output files

web-ifc-worker:
  volumes:
    - ifc_data:/data/ifc:ro   # Read-only: only reads IFC files

qgis-server:
  volumes:
    - ./gis-data:/data:ro     # Read-only: serves pre-prepared data

n8n:
  volumes:
    - ifc_data:/data/ifc      # Read-write: ingests new IFC files
```

ALWAYS use `:ro` (read-only) mount flag on services that only read IFC files. This prevents accidental corruption and makes data flow explicit.

---

## Anti-Pattern 7: Running ERPNext in the Same Compose Stack

### What Goes Wrong

```yaml
# WRONG -- ERPNext mixed into the AEC stack
services:
  ifcopenshell-worker:
    # ... AEC service
  erpnext:
    image: frappe/erpnext:v15
    depends_on:
      - mariadb
      - redis-erpnext
  mariadb:
    image: mariadb:10.6
  redis-erpnext:
    image: redis:7-alpine
  # Now you have TWO Redis instances, conflicting port needs, etc.
```

ERPNext requires its own MariaDB (not PostgreSQL), its own Redis instances (cache, queue, socketio), and multiple worker containers (bench-worker, scheduler). Mixing it with the AEC stack creates resource conflicts, port collisions, and operational complexity.

### Correct Approach

Run ERPNext as a separate deployment. Connect from n8n via the ERPNext REST API:

```yaml
# AEC stack: connect to ERPNext via HTTP
n8n:
  environment:
    ERPNEXT_URL: "https://erp.example.com"
    ERPNEXT_API_KEY: "${ERPNEXT_API_KEY}"
    ERPNEXT_API_SECRET: "${ERPNEXT_API_SECRET}"
```

NEVER deploy ERPNext in the same docker-compose as the AEC processing stack. ALWAYS connect via REST API.

---

## Anti-Pattern 8: Skipping Healthchecks and Startup Order

### What Goes Wrong

```yaml
# WRONG -- no healthchecks, no dependency conditions
ifcopenshell-worker:
  depends_on:
    - postgres
    - redis
```

`depends_on` without a condition only waits for the container to START, not for the service inside to be READY. The IfcOpenShell worker starts before PostgreSQL accepts connections, causing connection errors and crashes on first request.

### Correct Approach

```yaml
# CORRECT -- healthchecks with conditions
postgres:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U aec"]
    interval: 10s
    timeout: 5s
    retries: 5

ifcopenshell-worker:
  depends_on:
    postgres:
      condition: service_healthy
    redis:
      condition: service_healthy
```

ALWAYS add healthchecks to infrastructure services. ALWAYS use `condition: service_healthy` in `depends_on` blocks.

---

## Anti-Pattern 9: Using `latest` Tag for Critical AEC Images

### What Goes Wrong

```yaml
# WRONG -- unpinned versions
ifcopenshell-worker:
  image: continuumio/miniconda3:latest
  # Next build may pull a breaking change

qgis-server:
  image: qgis/qgis-server:latest
  # May jump from 3.34 to 3.36 with API changes
```

AEC tools have complex interdependencies. An unpinned QGIS Server image may update to a version that changes WMS behavior. An unpinned miniconda image may break conda-forge package resolution.

### Correct Approach

```yaml
# CORRECT -- pinned versions
ifcopenshell-worker:
  image: continuumio/miniconda3:24.7.1-0

qgis-server:
  image: qgis/qgis-server:ltr    # LTR tag is acceptable -- it tracks the long-term release

web-ifc-worker:
  # Pin web-ifc version in package.json
  # npm install web-ifc@0.0.57
```

ALWAYS pin base image versions for build reproducibility. The `:ltr` tag for QGIS Server is acceptable because it tracks a stable, tested release line.

---

## Anti-Pattern 10: Ignoring Speckle Server Platform Requirements

### What Goes Wrong

```yaml
# WRONG -- running on ARM64 Mac (Apple Silicon)
speckle-server:
  image: speckle/speckle-server
  # Fails: no ARM64 image available
```

Speckle Server Docker images are built ONLY for `linux/amd64`. Running them on ARM64 (Apple Silicon Macs, AWS Graviton) fails with exec format errors or emulation crashes.

### Correct Approach

```yaml
# CORRECT -- explicit platform declaration
speckle-server:
  image: speckle/speckle-server
  platform: linux/amd64
```

ALWAYS add `platform: linux/amd64` to all Speckle Server services. On ARM64 hosts, Docker Desktop will use Rosetta/QEMU emulation (with a ~20% performance penalty).
