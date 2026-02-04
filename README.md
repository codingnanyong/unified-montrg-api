# Unified Monitoring API

[![Python](https://img.shields.io/badge/Python-3.10+-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115.0-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-2.0.34-1C1C1C?logo=sqlalchemy&logoColor=white)](https://www.sqlalchemy.org/)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-Deployed-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Supported-336791?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![MySQL](https://img.shields.io/badge/MySQL-Supported-4479A1?logo=mysql&logoColor=white)](https://www.mysql.com/)

A FastAPI-based monitoring API skeleton for Kubernetes clusters.

## Quick Start

```bash
# Install dependencies (using Poetry)
poetry install

# Run the application
poetry run uvicorn app.main:app --reload

# Or with requirements.txt and virtualenv
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

**Local development:** Create a `.env` file in the project root to override settings when running locally.  
**Kubernetes deployment:** Database connection details are managed via Secrets. Configure them in `k8s/secret.yaml`. Do not commit this file to Git.

## Tests

```bash
poetry run pytest
```

## Docker Image Build

### Install Oracle Instant Client (for Oracle Database Connection)

Oracle Instant Client is required to connect to the Oracle database.

1. **Download Oracle Instant Client**
   - Oracle downloads: [https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html]
   - Download the **Instant Client Basic** package (e.g. `instantclient-basic-linux.x64-23.7.0.25.01.zip`)
   - An Oracle account may be required (free registration available)

2. **Build the Docker image**

   ```bash
   # Copy the Oracle Instant Client ZIP into the docker/ directory
   cp instantclient-basic-linux.x64-23.7.0.25.01.zip docker/instantclient-basic.zip
   
   # Build image (local tag: csg/unified-montrg-api:latest)
   docker build -f docker/Dockerfile -t csg/unified-montrg-api:latest .
   
   # Or specify the ZIP filename via build arg
   docker build -f docker/Dockerfile \
     --build-arg INSTANTCLIENT_ZIP=instantclient-basic-linux.x64-23.7.0.25.01.zip \
     -t csg/unified-montrg-api:latest .
   ```

3. **Verify the build**

   ```bash
   # (kubeadm) Load image into containerd
   docker save csg/unified-montrg-api:latest -o unified-montrg-api.tar
   sudo ctr -n k8s.io images import unified-montrg-api.tar
   
   # Run locally
   docker run --rm -p 8000:8000 --env-file .env csg/unified-montrg-api:latest
   ```

**Notes:**

- If the Oracle Instant Client ZIP is missing, the build will succeed but may show warnings
- The image may fall back to thin mode; some Oracle server versions do not support thin mode
- For CMMS database connectivity, include Oracle Instant Client in the build

Patterns defined in `docker/.dockerignore` are applied to the build context via the root `.dockerignore` symlink.

## Kubernetes Deployment

### Prerequisites

1. **Secrets:** Store database connection strings in Kubernetes Secrets.

```bash
# Create secret.yaml (see secret.example.yaml for reference)
# Set Base64-encoded database URLs
kubectl apply -f k8s/secret.yaml
```

Set the following in `k8s/secret.yaml` (Base64-encoded):

- `[machine]_DATABASE_URL`: `mysql+aiomysql://<USER>:<PASSWORD>@<HOST>:<PORT>/<DB>`
- `[service]_DATABASE_URL`: `postgresql+asyncpg://<USER>:<PASSWORD>@<HOST>:<PORT>/<DB>`

**Dynamic database support:**

- Environment variables matching `IP*_DATABASE_URL` are detected automatically
- Example: `MACH01_DATABASE_URL` → `MACH01`, `MACH02_DATABASE_URL` → `MACH02`, etc.
- Add new IPs by adding `MACH**_DATABASE_URL` entries to the Secret

### Deployment Script (Recommended)

Use `scripts/deploy.sh` to automate Docker image build and Kubernetes deployment.

```bash
cd ~/apps/unified_montrg
./scripts/deploy.sh
```

The script:

1. Builds the Docker image
2. Deploys the image to all nodes (containerd, for high availability)
3. Applies Kubernetes resources (Namespace → Secret → ConfigMap → Service → Deployment → Ingress Controller → Ingress)
4. Restarts the Deployment
5. Checks Pod status

**High availability:**

- Application Pods run with **5 replicas** (default)
- Ingress Controller is deployed as a **DaemonSet** on all nodes
- Traffic is **load-balanced** regardless of which node IP is used
- Service continues across node or Pod failures

**Customize via environment variables:**

```bash
IMAGE_NAME=my-registry/unified-montrg:1.0.0 \
NAMESPACE=my-namespace \
./scripts/deploy.sh
```

### Manual Deployment

Apply manifests in this order:

```bash
kf=~/apps/unified_montrg/k8s

# 1. Namespace
kubectl apply -f $kf/namespace.yaml

# 2. Secret (database credentials)
kubectl apply -f $kf/secret.yaml

# 3. ConfigMap (application settings)
kubectl apply -f $kf/configmap.yaml

# 4. Service (ClusterIP)
kubectl apply -f $kf/service.yaml

# 5. Deployment
kubectl apply -f $kf/deployment.yaml

# 6. Ingress Controller (DaemonSet)
kubectl apply -f $kf/ingress-controller-daemonset.yaml
kubectl apply -f $kf/ingress-controller-service-patch.yaml

# 7. Ingress
kubectl apply -f $kf/ingress.yaml
```

### Access

The API is exposed via the **Ingress Controller (DaemonSet)**. The controller runs on every node and **load-balances** traffic regardless of which node IP you use.

**High availability layout:**

- Ingress Controller runs as a DaemonSet on **all nodes**
- NodePort 30081 is available on every node
- Traffic is distributed across multiple Pods

**URLs:**

- **HTTP**: `http://<node1-ip>:30081/api` or `http://<node2-ip>:30081/api` (either node works)
- **API docs**: `http://<node-ip>:30081/api/docs`
- **ReDoc**: `http://<node-ip>:30081/api/redoc`

**Examples:**

```bash
# Use any node IP in the cluster
http://{Control Plane IP}:30081/api     # control-plane node
http://{Worker Node IP}}:30081/api     # worker node (same behavior)
```

### API Endpoints

- **IP Data API**: `/api/v1/ip-data/{pid}`
  - Use `db` query parameter to select database (ip04, ip12, ip20, ip34, ip37, etc.)
  - Example: `/api/v1/ip-data/12345?db=ip04&limit=100`
  
- **Machines API**: `/api/v1/machines`
  - Hierarchical machine information
  - Filters: `plant_cd`, `process_name`, `op_cd`, `line_no`, `machine_no`
  - Example: `/api/v1/machines?plant_cd=3120&process_name=IP/IU`

- **Health Check**: `/api/healthz`

### Deployment Status

```bash
# Pods
kubectl get pods -n unified-montrg

# Deployment
kubectl get deployment -n unified-montrg

# Services and Ingress
kubectl get svc,ingress -n unified-montrg

# Logs
kubectl logs -f deployment/unified-montrg -n unified-montrg

# Restart (after code changes)
kubectl rollout restart deployment/unified-montrg -n unified-montrg
```
