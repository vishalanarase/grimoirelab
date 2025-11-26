# Demo Overlay

This overlay spins up a full GrimoireLab stack in the `demo` namespace on AKS, complete with HTTPS
termination at the `grimoirelab-gateway` ingress and workload-to-workload communication inside the
cluster.

## Prerequisites

- `kubectl` configured against the target AKS cluster.
- Required secrets created in `demo`:
  - `grimoirelab-secrets` (SortingHat, MariaDB, etc.).
  - `netsuite-sync-secrets` if the NetSuite CronJob is enabled.
- The repository checked out with these manifests.

## Deploy

```bash
kubectl apply -f manifests/demo/namespace.yaml
kubectl apply -k manifests/demo
```

The overlay includes a self-signed TLS certificate (`grimoirelab-gateway-tls`) that already matches
the in-cluster DNS names. Regenerate it if you rotate domains or need a longer validity period.

Check status:

```bash
kubectl get pods -n demo
kubectl get svc -n demo
```

All expected Pods should report `1/1 Running` once MariaDB, OpenSearch, SortingHat, and Mordred are
healthy.

## Access

- **SortingHat UI** – served through the `grimoirelab-gateway` LoadBalancer. Use either
  `https://<LB-IP>/identities/` or `http://<LB-IP>/identities/`. Update the
  `SORTINGHAT_ALLOWED_HOST` env var (in `sortinghat.yaml`) if the LoadBalancer IP changes.
  Browsers will warn about the self-signed cert; accept to continue.
- **OpenSearch Dashboards** – exposed via the same LoadBalancer at `https://<LB-IP>/` (over the
  Kibana path) after NGINX proxies to the internal service.

## Notes

- Internal components (Mordred, SortingHat worker, NetSuite sync) communicate with SortingHat over
  HTTP on the gateway’s port 80, so their pods no longer mount the TLS secret.
- The `grimoirelab-gateway` Deployment listens on ports 80 and 443; the Service publishes both.
- If you reissue the TLS cert, update `manifests/demo/gateway-tls-secret.yaml` and reapply.

For troubleshooting, inspect pod logs, e.g. `kubectl logs deployment/mordred -n demo`, and ensure
the required secrets exist in the namespace.

