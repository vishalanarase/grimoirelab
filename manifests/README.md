# GrimoireLab Kubernetes Manifests

This directory contains the Kubernetes specs for running the GrimoireLab stack
via Kustomize. It mirrors the docker-compose topology (MariaDB, Valkey,
OpenSearch + Dashboards, SortingHat API/worker, Mordred, and an Nginx gateway)
and is organized into reusable bases and overlays.

## Layout

| Path | Description |
|------|-------------|
| `prod/` | Base manifests intended for remote/production clusters |
| `local/` | KIND overlay and helper files for workstation smoke-tests |

Every folder includes its own `kustomization.yaml`, so you can `kubectl apply -k`
or `kubectl delete -k` without touching individual files.

## Requirements

1. Kubernetes 1.24+ and `kubectl` access.
2. Storage classes that fulfil the PVCs defined in `prod/storage.yaml`
   (`ReadWriteOnce` for DB/cache/search data plus `ReadWriteMany` for the
   shared SortingHat static assets).
3. Nodes configured with `vm.max_map_count=262144` before scheduling OpenSearch.
4. Docker + KIND only if you intend to use the local overlay.

## Deploy to a cluster (`prod/`)

```bash
cd /Users/vishal/worktest/grimoirelab

# Namespace + secrets (replace example values before shared deployments)
kubectl apply -f manifests/prod/namespace.yaml
kubectl apply -f manifests/prod/secrets.example.yaml   # or kubectl create secret …

# Deploy the stack
kubectl apply -k manifests/prod
```

Key files in `prod/`:

- `configmaps.yaml` – embeds the default `setup.cfg`, `projects.json`,
  and Nginx templates from `default-grimoirelab-settings`.
- `storage.yaml` – PVCs matching the docker-compose volumes.
- `secrets.example.yaml` – documents the required keys; copy and customize for
  anything beyond local testing.
- `mariadb.yaml`, `opensearch.yaml`, `sortinghat.yaml`, and `mordred.yaml`
  include all fixes needed for production (empty MariaDB password handling,
  authenticated OpenSearch probes, routing SortingHat traffic through
  `grimoirelab-gateway`, etc.), so `kubectl apply -k manifests/prod` works
  as-is on any Kubernetes cluster that meets the requirements above.
- `netsuite-sync-cronjob.yaml` – optional weekly CronJob that syncs
  organizations/users from NetSuite into SortingHat once the accompanying Secret
  (`netsuite-sync-secrets.example.yaml`) is populated.

## Local KIND overlay (`local/`)

The overlay keeps the same workloads but adapts the plumbing:

- `kind-cluster.yaml` exposes host port 8000 and bind-mounts `/tmp/grimoirelab`.
- `storage-hostpath.yaml` and `sortinghat-static-pvc.yaml` provide an RWX host
  path for SortingHat static files so Nginx can serve them too.
- `nginx-nodeport.yaml` flips the gateway Service to `NodePort` 30080.

Quick start:

```bash
mkdir -p /tmp/grimoirelab
kind create cluster --name grimoirelab --config manifests/local/kind-cluster.yaml
kubectl apply -f manifests/prod/namespace.yaml
kubectl apply -f manifests/prod/secrets.example.yaml   # or create your own Secret
kubectl apply -k manifests/local
```

See `manifests/local/README.md` for the full workflow and teardown commands.

## Customization & verification

- Update `configmaps.yaml` whenever you change `setup.cfg` or `projects.json`.
- Replace inline secrets with references to Kubernetes Secrets or your preferred
  secret manager.
- Tweak resource requests/limits or add node selectors/taints per workload to
  fit your cluster.
- Swap the gateway Service type in `prod/nginx.yaml` if LoadBalancer IPs are not
  available (NodePort + Ingress works well).

Basic health checks:

```bash
kubectl get pods -n grimoirelab
kubectl logs deploy/mordred -n grimoirelab -f
kubectl port-forward svc/opensearch-node1 -n grimoirelab 9200:9200
curl -k https://admin:<password>@localhost:9200/_cluster/health
```

If every pod reports `1/1 Running` and the Mordred logs show data
collection progress, the production stack is ready. If you redeploy with
`kubectl apply -k manifests/prod`, no additional manual tweaks are
required—the manifests already encode the fixes applied during testing.

Delete the stack with `kubectl delete -k manifests/prod` (PVCs remain until you
remove them manually). Iterate locally with the KIND overlay, then promote the
same base manifests to your target cluster.

## NetSuite → SortingHat sync (optional)

1. Build and push the helper image (from `netsuite-sync-service/`):
   ```bash
   cd /Users/vishal/worktest/netsuite-sync-service
   docker build -t ghcr.io/<org>/netsuite-sync-service:latest .
   docker push ghcr.io/<org>/netsuite-sync-service:latest
   ```
2. Create the Secret with your SortingHat + NetSuite credentials. Use
   `netsuite-sync-secrets.example.yaml` as a template, then `kubectl apply -f`.
3. Deploy the CronJob (already part of `kubectl apply -k manifests/prod`). By
   default it runs every Monday at 03:00 UTC, calling the internal SortingHat
   API via `grimoirelab-gateway`.

Adjust the schedule, organization name, or image tag inside
`netsuite-sync-cronjob.yaml` as needed for your environment.***

