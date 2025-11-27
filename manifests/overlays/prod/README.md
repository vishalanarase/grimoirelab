# Production Overlay

This overlay deploys the GrimoireLab stack in the `prod` namespace for production use.
It uses Azure-managed storage classes and is designed for AKS clusters.

## Prerequisites

- `kubectl` configured against the target AKS cluster.
- Required secrets created in `prod` namespace:
  - `grimoirelab-secrets` (SortingHat, MariaDB, OpenSearch credentials).
  - `netsuite-sync-secrets` if the NetSuite CronJob is enabled.
- Storage classes: `managed-premium` (for RWO) and `azurefile-csi` (for RWX).
- Nodes configured with `vm.max_map_count=262144` for OpenSearch.

## Deploy

```bash
# Create secrets first
kubectl create secret generic grimoirelab-secrets \
  --namespace prod \
  --from-literal=mysql-root-password='<secure-password>' \
  --from-literal=sortinghat-db-password='<secure-password>' \
  --from-literal=sortinghat-superuser-password='<secure-password>' \
  --from-literal=sortinghat-secret-key='<secure-random-key>' \
  --from-literal=opensearch-initial-admin-password='<secure-password>'

# Deploy the stack
kubectl apply -k manifests/overlays/prod
```

Check status:

```bash
kubectl get pods -n prod
kubectl get svc -n prod
```

All expected Pods should report `1/1 Running` once MariaDB, OpenSearch, SortingHat, and Mordred are
healthy.

## Access

- **SortingHat UI** – served through the `grimoirelab-gateway` LoadBalancer on port 8000.
  Access at `http://<LB-IP>:8000/identities/`
- **OpenSearch Dashboards** – exposed via the same LoadBalancer at `http://<LB-IP>:8000/`

## Configuration

### Storage

The overlay uses Azure storage classes:
- `managed-premium` - For MariaDB, Valkey, and OpenSearch data (ReadWriteOnce)
- `azurefile-csi` - For SortingHat static files (ReadWriteMany)

### Networking

The nginx gateway listens on port 8000 (HTTP only) and is exposed via LoadBalancer.
If you need HTTPS, consider adding Azure Application Gateway or updating the nginx configuration.

### Projects Configuration

Edit `projects.json` in this overlay to configure which repositories to track:

```json
{
  "ProjectName": {
    "git": ["https://github.com/org/repo.git"],
    "github": ["https://github.com/org/repo"]
  }
}
```

After editing, reapply:

```bash
kubectl apply -k manifests/overlays/prod
kubectl rollout restart deployment/mordred -n prod
```

## NetSuite Sync

To enable NetSuite synchronization:

1. Create the netsuite-sync-secrets:
   ```bash
   kubectl apply -f netsuite-sync-secrets.example.yaml -n prod
   ```
2. The CronJob is already included and runs every Monday at 03:00 UTC.

## Troubleshooting

View logs:
```bash
kubectl logs deployment/mordred -n prod -f
kubectl logs deployment/sortinghat -n prod
kubectl logs deployment/opensearch-node1 -n prod
```

Check resource status:
```bash
kubectl get all -n prod
kubectl describe pod <pod-name> -n prod
```

## Cleanup

```bash
kubectl delete -k manifests/overlays/prod
```

Note: PVCs are not automatically deleted. Remove them manually if needed:

```bash
kubectl delete pvc -n prod --all
```

