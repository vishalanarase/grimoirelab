# GrimoireLab Kubernetes Manifests

This directory contains the Kubernetes specs for running the GrimoireLab stack
via Kustomize. It mirrors the docker-compose topology (MariaDB, Valkey,
OpenSearch + Dashboards, SortingHat API/worker, Mordred, and an Nginx gateway)
and is organized following Kustomize best practices with a base and overlays.

## Layout

| Path | Description |
|------|-------------|
| `base/` | Common base manifests with core resources grouped by type |
| `overlays/local/` | KIND overlay for local development and testing |
| `overlays/prod/` | Production overlay for remote clusters |
| `overlays/demo/` | Demo overlay for AKS deployment |

Every folder includes its own `kustomization.yaml`, so you can `kubectl apply -k`
or `kubectl delete -k` without touching individual files.

## Base Structure

The `base/` directory contains common resources organized by type:

- `namespace/` - Namespace definition
- `config/` - ConfigMaps for application configuration
- `storage/` - PersistentVolumeClaims
- `databases/` - MariaDB and Valkey (Redis)
- `opensearch/` - OpenSearch and OpenSearch Dashboards
- `apps/` - SortingHat and Mordred applications
- `networking/` - Nginx gateway and network policies
- `cronjobs/` - NetSuite sync CronJob

## Requirements

1. Kubernetes 1.24+ and `kubectl` access.
2. Storage classes that fulfil the PVCs defined in `base/storage/`
   (`ReadWriteOnce` for DB/cache/search data plus `ReadWriteMany` for the
   shared SortingHat static assets).
3. Nodes configured with `vm.max_map_count=262144` before scheduling OpenSearch.
4. Docker + KIND only if you intend to use the local overlay.

## Quick Start

### Local Development (KIND)

```bash
# Create local directory for volumes
mkdir -p /tmp/grimoirelab

# Create KIND cluster
kind create cluster --name grimoirelab --config manifests/overlays/local/kind-cluster.yaml

# Create secrets
kubectl create secret generic grimoirelab-secrets \
  --namespace grimoirelab \
  --from-literal=mysql-root-password='local-root' \
  --from-literal=sortinghat-db-password='local-root' \
  --from-literal=sortinghat-superuser-password='admin' \
  --from-literal=sortinghat-secret-key='change-me' \
  --from-literal=opensearch-initial-admin-password='GrimoireLab.1'

# Deploy the stack
kubectl apply -k manifests/overlays/local
```

Access the stack at `http://localhost:8000/`

See `manifests/overlays/local/README.md` for detailed instructions.

### Production Deployment

```bash
# Deploy to production cluster
kubectl apply -k manifests/overlays/prod
```

See `manifests/overlays/prod/README.md` for detailed instructions.

### Demo Deployment (AKS)

```bash
# Deploy to demo environment
kubectl apply -k manifests/overlays/demo
```

See `manifests/overlays/demo/README.md` for detailed instructions.

## Customization

Each overlay can be customized by:

1. **ConfigMaps**: Edit `setup.cfg` and `projects.json` in the overlay directory
2. **Secrets**: Create environment-specific secrets (see `secrets.example.yaml`)
3. **Storage**: Adjust storage classes in `storage-patches.yaml`
4. **Resources**: Add resource requests/limits via patches
5. **Networking**: Modify service types and ingress configuration

## Verification

Basic health checks:

```bash
# Check pod status
kubectl get pods -n <namespace>

# View Mordred logs
kubectl logs deploy/mordred -n <namespace> -f

# Check OpenSearch health
kubectl port-forward svc/opensearch-node1 -n <namespace> 9200:9200
curl -k https://admin:<password>@localhost:9200/_cluster/health
```

If every pod reports `1/1 Running` and the Mordred logs show data
collection progress, the stack is ready.

## NetSuite → SortingHat sync (optional)

1. Build and push the helper image (from `netsuite-sync-service/`):
   ```bash
   docker build -t ghcr.io/<org>/netsuite-sync-service:latest .
   docker push ghcr.io/<org>/netsuite-sync-service:latest
   ```
2. Create the Secret with your SortingHat + NetSuite credentials:
   ```bash
   kubectl apply -f manifests/overlays/<env>/netsuite-sync-secrets.example.yaml
   ```
3. Deploy the CronJob (already part of `kubectl apply -k`). By
   default it runs every Monday at 03:00 UTC.

## Additional Documentation

- `manifests/overlays/local/README.md` - Local KIND deployment guide
- `manifests/overlays/demo/README.md` - Demo environment guide
- `manifests/overlays/demo/README.dns.md` - DNS configuration guide
- `manifests/README.azure.md` - Azure-specific deployment notes
