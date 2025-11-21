# Local KIND Deployment

Use this overlay to spin up the GrimoireLab stack on a single-node
[kind](https://kind.sigs.k8s.io/) cluster before promoting to another
environment. It keeps the base Kubernetes manifests but tweaks the
ingress/volume wiring to match the Docker-in-Docker runtime.

## Layout

| File | Purpose |
|------|---------|
| `kind-cluster.yaml` | Sample kind config with port + volume mappings |
| `kustomization.yaml` | Overlay that reuses the base manifests |
| `storage-hostpath.yaml` | HostPath PV backing the SortingHat static assets |
| `sortinghat-static-pvc.yaml` | Forces the PVC to bind to the HostPath PV |
| `nginx-nodeport.yaml` | Switches the gateway Service to NodePort 30080 |

## Cluster prep

1. Ensure `/tmp/grimoirelab` exists on your host; this is bind-mounted into
   the kind node so that the HostPath PV survives pod restarts:
   ```bash
   mkdir -p /tmp/grimoirelab
   ```
2. Create (or reuse) a kind cluster with the provided config:
   ```bash
   kind create cluster --name grimoirelab --config manifests/local/kind-cluster.yaml
   ```
   This exposes port `8000` on your host and pre-mounts `/tmp/grimoirelab`.

## Deploy

1. Populate credentials (same as base instructions). For quick tests:
   ```bash
   kubectl apply -f manifests/prod/namespace.yaml
   kubectl create secret generic grimoirelab-secrets \
     --namespace grimoirelab \
     --from-literal=mysql-root-password='local-root' \
     --from-literal=sortinghat-db-password='local-root' \
     --from-literal=sortinghat-superuser-password='admin' \
     --from-literal=sortinghat-secret-key='change-me' \
     --from-literal=opensearch-initial-admin-password='GrimoireLab.1'
   ```
2. Apply the overlay:
   ```bash
   kubectl apply -k manifests/local
   ```
   (This automatically pulls in the base manifests.)

## What changed vs. base

- The Nginx `Service` is `NodePort` (30080) and kind forwards it to host
  port `8000`, so you can browse `http://localhost:8000/`.
- The SortingHat static content uses a `HostPath` PV (`/tmp/grimoirelab/sortinghat-static`)
  to mimic the original shared volume between SortingHat and Nginx without
  requiring an RWX-capable CSI driver.
- All other PVCs continue to use the default kind storage class.

## Cleanup

```bash
kubectl delete -k manifests/local
kind delete cluster --name grimoirelab
rm -rf /tmp/grimoirelab/sortinghat-static
```

You can now iterate locally, inspect logs via `kubectl logs`, and once
ready, deploy the base manifests to your real cluster.
