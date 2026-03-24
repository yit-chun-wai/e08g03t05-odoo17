# Deployment Guide — Odoo 17 on Azure Kubernetes Service (AKS)

## Prerequisites
- Docker Desktop installed
- `kubectl` installed
- Azure CLI (`az`) installed
- Access to the Azure subscription
- Access to Docker Hub account (`jiaying0811`)

---

## Running Locally (Docker)

### First time setup
```bash
git pull
docker compose down -v     # wipe old database
docker compose up          # auto-restores database from odoo.sql.gz
```

Access the app at `http://localhost:8069`

### Updating local database (after making changes in the app)
```bash
# Export latest database
docker compose exec db pg_dump -U odoo esm-odoo | gzip > odoo.sql.gz

# Push to GitHub
git add odoo.sql.gz
git commit -m "update database dump"
git push
```

### Teammates pulling latest data
```bash
git pull
docker compose down -v
docker compose up
```

---

## Kubernetes Deployment (AKS)

### Architecture
| Component | Details |
|---|---|
| Cloud Provider | Microsoft Azure (Azure for Students) |
| Kubernetes Cluster | AKS — `odoo17-aks` in resource group `odoo17-rg` |
| Node | 1x Standard_D2as_v4 (2 vCPU, 8GB RAM) |
| Docker Image | `jiaying0811/odoo17:latest` (multi-platform: amd64 + arm64) |
| Database | PostgreSQL 13 — database name: `esm-odoo` |
| Public URL | `http://85.211.217.93` |

### Kubernetes Files
```
k8s/
├── namespace.yaml      # creates 'odoo' namespace
├── postgres.yaml       # PostgreSQL PVC + Deployment + Service
└── odoo.yaml           # Odoo PVC + Deployment + LoadBalancer Service
```

---

## First-Time AKS Deployment

### 1. Connect to the cluster
```bash
az login
az aks get-credentials --resource-group odoo17-rg --name odoo17-aks
```

### 2. Apply Kubernetes manifests
```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/odoo.yaml
```

### 3. Wait for pods to be ready
```bash
kubectl get pods -n odoo
```
Both `odoo` and `postgres` pods should show `1/1 Running`.

### 4. Restore the database
Get the postgres pod name from the previous step, then:
```bash
# Copy the dump into the postgres pod
kubectl cp odoo.sql.gz odoo/<postgres-pod-name>:/tmp/odoo.sql.gz

# Restore it
kubectl exec -n odoo <postgres-pod-name> -- bash -c "gunzip -c /tmp/odoo.sql.gz | psql -U odoo -d esm-odoo"
```

### 5. Restore the filestore
Get the odoo pod name, then:
```bash
kubectl cp filestore\esm-odoo\. odoo/<odoo-pod-name>:/var/lib/odoo/filestore/esm-odoo -n odoo
kubectl exec -it <odoo-pod-name> -n odoo -- chown -R odoo:odoo /var/lib/odoo/filestore
```

### 6. Update the base URL in the database
```bash
kubectl exec -n odoo <postgres-pod-name> -- psql -U odoo -d esm-odoo -c \
  "UPDATE ir_config_parameter SET value='http://85.211.217.93' WHERE key='web.base.url';"
```

### 7. Restart Odoo
```bash
kubectl rollout restart deployment/odoo -n odoo
```

Access the app at `http://85.211.217.93`

---

## Updating the Docker Image

When code changes are made, rebuild and push the image from Mac:
```bash
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t jiaying0811/odoo17:latest --push .
```

Then redeploy on the cluster:
```bash
kubectl rollout restart deployment/odoo -n odoo
```

---

## Useful Commands

```bash
# Check pod status
kubectl get pods -n odoo

# Check service and public IP
kubectl get svc -n odoo

# View Odoo logs
kubectl logs deployment/odoo -n odoo --tail=50

# View Postgres logs
kubectl logs deployment/postgres -n odoo --tail=50

# Restart deployments
kubectl rollout restart deployment/odoo -n odoo
kubectl rollout restart deployment/postgres -n odoo
```

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| `ErrImagePull` | Image is ARM64 only | Rebuild with `buildx` for `linux/amd64,linux/arm64` |
| `CrashLoopBackOff` on postgres | `lost+found` in PVC mount | `PGDATA` env var is set to use a subdirectory |
| `PermissionError` on `/var/lib/odoo/sessions` | PVC mounted as root | Init container fixes permissions on startup |
| 500 error / no CSS | Empty database or missing filestore | Restore `odoo.sql.gz` and copy filestore |
| No styling / broken images | `web.base.url` points to localhost | Update `web.base.url` in database to the public IP |
