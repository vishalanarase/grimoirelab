# Azure Kubernetes Service (AKS) Deployment Guide

This guide walks you through deploying GrimoireLab on Azure Kubernetes Service (AKS).

## Prerequisites

- AKS cluster (≥1.26) with nodes offering at least 4 vCPU, 12 GiB RAM
- `kubectl` configured against your AKS cluster
- Azure Container Registry (ACR) or another registry you can push to
- Azure CLI installed and configured (`az login`)
- `kubectl` access to your cluster

## Step-by-Step Deployment

### 1. Prepare the Cluster

#### Set `vm.max_map_count` for OpenSearch

OpenSearch requires `vm.max_map_count` to be set to at least 262144. Apply this DaemonSet:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: sysctl
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: sysctl
  template:
    metadata:
      labels:
        name: sysctl
    spec:
      hostNetwork: true
      hostPID: true
      initContainers:
      - name: sysctl
        image: busybox:1.36
        securityContext:
          privileged: true
        command:
        - sh
        - -c
        - |
          sysctl -w vm.max_map_count=262144
          while true; do sleep 86400; done
      containers:
      - name: pause
        image: k8s.gcr.io/pause
EOF
```

Or manually set on each node if you have SSH access:
```bash
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -w vm.max_map_count=262144
```

#### Verify Storage Classes

Check available storage classes:
```bash
kubectl get storageclass
```

You should see:
- `managed-premium` or `managed-csi` (for ReadWriteOnce volumes)
- `azurefile-csi` (for ReadWriteMany volumes)

If `azurefile-csi` is missing, enable it:
```bash
az aks update -n <cluster-name> -g <resource-group> --enable-azurefile-csi-driver
```

### 2. Update Storage Configuration

The `storage.yaml` has been updated with Azure storage classes:
- **RWO volumes** (mariadb, valkey, opensearch): `managed-premium`
- **RWX volume** (sortinghat-static): `azurefile-csi`

If your cluster uses different storage class names, update `manifests/prod/storage.yaml` accordingly.

### 3. Build and Push Container Images

#### Option A: Use Public Images (Quick Start)

If you want to use public Docker Hub images, skip this step. The manifests already reference:
- `grimoirelab/grimoirelab:latest`
- `grimoirelab/sortinghat:latest`
- `grimoirelab/sortinghat-worker:latest`

#### Option B: Use Azure Container Registry (Recommended for Production)

1. **Login to ACR:**
   ```bash
   az acr login --name <acr-name>
   ```

2. **Build and push NetSuite sync service:**
   ```bash
   cd /path/to/netsuite-sync-service
   docker build -t <acr-name>.azurecr.io/netsuite-sync-service:prod .
   docker push <acr-name>.azurecr.io/netsuite-sync-service:prod
   ```

3. **Update netsuite-sync-cronjob.yaml:**
   ```yaml
   image: <acr-name>.azurecr.io/netsuite-sync-service:prod
   ```

4. **(Optional) Mirror GrimoireLab images to ACR:**
   ```bash
   az acr import --name <acr-name> --source docker.io/grimoirelab/grimoirelab:latest --image grimoirelab/grimoirelab:latest
   az acr import --name <acr-name> --source docker.io/grimoirelab/sortinghat:latest --image grimoirelab/sortinghat:latest
   az acr import --name <acr-name> --source docker.io/grimoirelab/sortinghat-worker:latest --image grimoirelab/sortinghat-worker:latest
   ```

5. **If using ACR, configure image pull secrets:**
   ```bash
   kubectl create secret docker-registry acr-secret \
     --docker-server=<acr-name>.azurecr.io \
     --docker-username=<service-principal-id> \
     --docker-password=<service-principal-password> \
     --namespace grimoirelab
   ```

   Then add to deployments (update each deployment's `spec.template.spec`):
   ```yaml
   imagePullSecrets:
     - name: acr-secret
   ```

### 4. Configure Secrets

#### Create GrimoireLab Secrets

1. **Copy and customize the secrets file:**
   ```bash
   cp manifests/prod/secrets.example.yaml manifests/prod/secrets.yaml
   ```

2. **Edit `secrets.yaml` with your credentials:**
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: grimoirelab-secrets
     namespace: grimoirelab
   type: Opaque
   stringData:
     mysql-root-password: "<strong-password>"
     sortinghat-db-password: "<strong-password>"
     sortinghat-superuser-password: "<admin-password>"
     sortinghat-secret-key: "<random-secret-key>"
     opensearch-initial-admin-password: "<strong-password>"
   ```

3. **Apply the secrets:**
   ```bash
   kubectl apply -f manifests/prod/secrets.yaml
   ```

#### Create NetSuite Sync Secrets (Optional)

If using NetSuite sync:

1. **Copy and customize:**
   ```bash
   cp manifests/prod/netsuite-sync-secrets.example.yaml manifests/prod/netsuite-sync-secrets.yaml
   ```

2. **Fill in your NetSuite OAuth credentials:**
   ```yaml
   stringData:
     sortinghat-username: admin
     sortinghat-password: "<from-grimoirelab-secrets>"
     netsuite-account-id: "<your-account-id>"
     netsuite-consumer-key: "<your-consumer-key>"
     netsuite-consumer-secret: "<your-consumer-secret>"
     netsuite-token: "<your-token>"
     netsuite-token-secret: "<your-token-secret>"
     netsuite-script-id: "<your-script-id>"
     netsuite-deploy-id: "1"
   ```

3. **Apply:**
   ```bash
   kubectl apply -f manifests/prod/netsuite-sync-secrets.yaml
   ```

### 5. Deploy the Stack

```bash
# Create namespace
kubectl apply -f manifests/prod/namespace.yaml

# Deploy all resources
kubectl apply -k manifests/prod
```

This will create:
- MariaDB (database)
- Valkey (Redis-compatible cache)
- OpenSearch (search engine)
- OpenSearch Dashboards (visualization)
- SortingHat API and Worker (identity management)
- Mordred (data collection and enrichment)
- Nginx Gateway (reverse proxy)
- NetSuite Sync CronJob (optional)

### 6. Verify Deployment

#### Check Pod Status

```bash
kubectl get pods -n grimoirelab
```

All pods should show `1/1 Running` or `Completed` status.

#### Check Services

```bash
kubectl get svc -n grimoirelab
```

Note the `EXTERNAL-IP` of `grimoirelab-gateway` (LoadBalancer service).

#### Check Logs

```bash
# Mordred (data collection)
kubectl logs -f deploy/mordred -n grimoirelab

# OpenSearch health
kubectl port-forward svc/opensearch-node1 -n grimoirelab 9200:9200 &
curl -k https://admin:<password>@localhost:9200/_cluster/health
```

### 7. Access the Application

#### Get LoadBalancer IP

```bash
kubectl get svc grimoirelab-gateway -n grimoirelab
```

#### Access URLs

- **OpenSearch Dashboards**: `http://<loadbalancer-ip>:8000/`
- **SortingHat API**: `http://<loadbalancer-ip>:8000/identities/api/`

#### Import Dashboards

1. Access OpenSearch Dashboards at `http://<loadbalancer-ip>:8000/`
2. Login with `admin` and your OpenSearch password
3. Go to **Stack Management → Saved Objects**
4. Click **Import** and upload the NDJSON files from `panels/ndjson/`:
   - `git.ndjson`
   - `mirantis.ndjson`
   - `upstream-contribution-tracking.ndjson`
5. After import, update index patterns to match your indices:
   - `git` → `git_demo_enriched`
   - `git_areas_of_code` → `git-aoc_demo_enriched`

### 8. Configure Ingress (Optional)

For HTTPS and custom domains, replace the LoadBalancer service with an Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grimoirelab-ingress
  namespace: grimoirelab
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - grimoirelab.yourdomain.com
      secretName: grimoirelab-tls
  rules:
    - host: grimoirelab.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grimoirelab-gateway
                port:
                  number: 8000
```

### 9. Monitoring and Maintenance

#### Monitor Resource Usage

```bash
kubectl top pods -n grimoirelab
kubectl top nodes
```

#### Scale Resources (if needed)

```bash
# Scale Mordred
kubectl scale deployment mordred -n grimoirelab --replicas=2

# Scale SortingHat worker
kubectl scale deployment sortinghat-worker -n grimoirelab --replicas=2
```

#### Backup Data

```bash
# Backup MariaDB
kubectl exec -n grimoirelab deploy/mariadb -- mysqldump -uroot -p<password> sortinghat_db > backup.sql

# Backup OpenSearch (use snapshot API)
kubectl port-forward svc/opensearch-node1 -n grimoirelab 9200:9200
curl -X PUT "https://admin:<password>@localhost:9200/_snapshot/backup" -H 'Content-Type: application/json' -d'
{
  "type": "fs",
  "settings": {
    "location": "/usr/share/opensearch/backup"
  }
}'
```

#### Update Secrets

```bash
# Edit secret
kubectl edit secret grimoirelab-secrets -n grimoirelab

# Restart pods to pick up new secrets
kubectl rollout restart deployment -n grimoirelab
```

### 10. Troubleshooting

#### Pods Not Starting

```bash
# Check pod events
kubectl describe pod <pod-name> -n grimoirelab

# Check logs
kubectl logs <pod-name> -n grimoirelab
```

#### Storage Issues

```bash
# Check PVC status
kubectl get pvc -n grimoirelab

# Check storage classes
kubectl get storageclass
```

#### OpenSearch Not Ready

```bash
# Check OpenSearch logs
kubectl logs deploy/opensearch-node1 -n grimoirelab

# Verify vm.max_map_count
kubectl exec -n grimoirelab deploy/opensearch-node1 -- sysctl vm.max_map_count
```

#### No Data in Dashboards

1. Verify Mordred is collecting data: `kubectl logs deploy/mordred -n grimoirelab`
2. Check OpenSearch indices: `kubectl exec -n grimoirelab deploy/opensearch-node1 -- curl -k -u admin:<password> https://localhost:9200/_cat/indices?v`
3. Ensure index patterns are configured correctly in Dashboards

### 11. Cleanup

To remove the deployment:

```bash
# Delete all resources
kubectl delete -k manifests/prod

# Delete namespace (this will delete everything in the namespace)
kubectl delete namespace grimoirelab

# Note: PVCs are retained by default. To delete them:
kubectl delete pvc --all -n grimoirelab
```

## Summary of Changes Made

1. ✅ **storage.yaml**: Updated with Azure storage classes
   - `managed-premium` for RWO volumes
   - `azurefile-csi` for RWX volumes

2. ✅ **netsuite-sync-cronjob.yaml**: Added TODO comment for ACR image reference

3. ✅ **This guide**: Comprehensive deployment instructions

## Next Steps

- Configure monitoring (Azure Monitor, Prometheus)
- Set up automated backups
- Configure autoscaling
- Set up CI/CD for image updates
- Configure network policies for security

