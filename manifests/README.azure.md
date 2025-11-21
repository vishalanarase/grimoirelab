## Deploying GrimoireLab on Azure Kubernetes Service (AKS)

Follow these steps to run the full stack (MariaDB, Valkey, OpenSearch, SortingHat, Mordred, gateway, NetSuite sync CronJob) on a managed AKS cluster.

### 1. Prerequisites

- AKS cluster (≥1.26) with nodes offering at least 4 vCPU, 12 GiB RAM.
- `kubectl` configured against the cluster.
- Azure Container Registry (ACR) or another registry you can push to.
- Ability to set node sysctl values (for OpenSearch) or use a privileged DaemonSet.

### 2. Prepare the cluster

1. **Set `vm.max_map_count`** on every node (required by OpenSearch):
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/elastic/helm-charts/main/elasticsearch/examples/sysctl/sysctl.yaml
   ```
   (Or run the equivalent command manually on each VM.)

2. **Storage classes**
   - Update `manifests/prod/storage.yaml` with AKS classes:
     ```yaml
     storageClassName: managed-premium          # for RWO PVCs
     ...
     storageClassName: azurefile-csi           # for sortinghat-static RWX volume
     ```
   - Ensure `azurefile-csi` exists (default on modern AKS).

### 3. Publish container images

1. Login to ACR: `az acr login -n <acr-name>`
2. Build/push the NetSuite sync helper:
   ```bash
   cd /Users/vishal/worktest/netsuite-sync-service
   docker build -t <acr-name>.azurecr.io/netsuite-sync-service:prod .
   docker push <acr-name>.azurecr.io/netsuite-sync-service:prod
   ```
3. If you want to mirror the public GrimoireLab images into ACR, repeat for each service. Otherwise, ensure your cluster can pull from Docker Hub.
4. Update `manifests/prod/netsuite-sync-cronjob.yaml` to reference your ACR tag.

### 4. Configure secrets

1. Copy `manifests/prod/secrets.example.yaml` to a safe location, fill in real credentials (MariaDB root password, SortingHat admin, OpenSearch admin) and apply:
   ```bash
   kubectl apply -f manifests/prod/secrets.example.yaml
   ```
2. Do the same for `manifests/prod/netsuite-sync-secrets.example.yaml` with your NetSuite OAuth values:
   ```bash
   kubectl apply -f manifests/prod/netsuite-sync-secrets.example.yaml
   ```

### 5. Deploy the stack via Kustomize

```bash
cd /Users/vishal/worktest/grimoirelab
kubectl apply -f manifests/prod/namespace.yaml
kubectl apply -k manifests/prod
```

This applies all Deployments, Services, CronJobs, and PVCs in one shot.

### 6. Networking

- `manifests/prod/nginx.yaml` uses a `LoadBalancer` Service on port 8000. AKS will provision an Azure Standard LB automatically; note the public IP with:
  ```bash
  kubectl get svc grimoirelab-gateway -n grimoirelab
  ```
- Access SortingHat + OpenSearch Dashboards via `http://<lb-ip>:8000/`.
- If you need Ingress or HTTPS, replace the Service with an Ingress controlled by Azure Application Gateway or NGINX Ingress Controller.

### 7. Verification

```bash
kubectl get pods -n grimoirelab
kubectl logs deploy/mordred -n grimoirelab -f
kubectl port-forward svc/opensearch-node1 -n grimoirelab 9200:9200 &
curl -k https://admin:<OPENSEARCH_PASSWORD>@localhost:9200/_cluster/health
kubectl get cronjobs -n grimoirelab
```

Import dashboards through the gateway (Stack Management → Saved Objects). The default Dashboards config already selects the global tenant, so NDJSON imports should succeed.

### 8. NetSuite sync CronJob

- Confirm the Secret exists: `kubectl get secret netsuite-sync-secrets -n grimoirelab`
- Trigger a manual run for testing:
  ```bash
  kubectl create job --from=cronjob/netsuite-sync netsuite-sync-manual -n grimoirelab
  kubectl logs job/netsuite-sync-manual -n grimoirelab -f
  ```

### 9. Maintenance tips

- Rotate credentials by updating the Secrets and running `kubectl apply -k manifests/prod`.
- If you mirror images into ACR, ensure `imagePullSecrets` are set on the namespaces/workloads if needed.
- Monitor OpenSearch (port 9600) and Mordred logs for ingestion health.

With the storage class adjustments, secrets applied, and images hosted in your registry, `kubectl apply -k manifests/prod` works unchanged on AKS. Use this README as a quick reference for environment-specific tweaks.

