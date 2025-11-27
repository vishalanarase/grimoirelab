# DNS Domain Name Configuration Guide

This guide covers the changes required when migrating from a LoadBalancer IP address to a DNS domain name for the demo deployment.

## Overview

When you obtain a DNS domain name (e.g., `grimoirelab.example.com`), you need to update several configuration files to ensure:
- TLS certificates include the domain in Subject Alternative Names (SANs)
- SortingHat accepts requests from the domain
- NGINX serves requests for the domain
- CORS policies allow the domain (if needed)

## Required Changes

### 1. Regenerate TLS Certificate

**File:** `manifests/demo/gateway-tls-secret.yaml`

The certificate must include your domain name in the Subject Alternative Names. Regenerate it with:

```bash
# Generate CA key and certificate
openssl genrsa -out gateway-ca.key 2048
openssl req -x509 -new -key gateway-ca.key -sha256 -days 365 \
  -out gateway-ca.crt \
  -subj "/CN=grimoirelab-gateway-ca" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign"

# Generate server key
openssl genrsa -out gateway-server.key 2048

# Generate certificate signing request with domain
openssl req -new -key gateway-server.key -out gateway-server.csr \
  -subj "/CN=grimoirelab-gateway" \
  -addext "subjectAltName=DNS:grimoirelab-gateway,DNS:grimoirelab-gateway.demo.svc,DNS:grimoirelab-gateway.demo.svc.cluster.local,DNS:your-domain.com,DNS:*.your-domain.com"

# Sign the certificate
openssl x509 -req -in gateway-server.csr -CA gateway-ca.crt -CAkey gateway-ca.key \
  -CAcreateserial -out gateway-server.crt -days 365 -sha256 -copy_extensions copy

# Create certificate chain (server cert + CA cert)
cat gateway-server.crt gateway-ca.crt > gateway-chain.crt
```

Then update `gateway-tls-secret.yaml`:
- `tls.crt`: Use the certificate chain (`gateway-chain.crt`)
- `tls.key`: Use `gateway-server.key`
- `ca.crt`: Use `gateway-ca.crt`

**Important:** Replace `your-domain.com` with your actual domain name in the SAN list.

### 2. Update SortingHat Allowed Hosts

**File:** `manifests/demo/sortinghat.yaml`

Update the `SORTINGHAT_ALLOWED_HOST` environment variable to include your domain:

```yaml
- name: SORTINGHAT_ALLOWED_HOST
  value: sortinghat,grimoirelab-gateway,localhost,127.0.0.1,[::1],your-domain.com,*.your-domain.com
```

Replace `your-domain.com` with your actual domain. Include both the base domain and wildcard if you use subdomains.

### 3. Update NGINX server_name

**File:** `manifests/demo/configmaps.yaml`

In the `nginx.conf.template` section, update the `server_name` directive:

```nginx
server_name localhost nginx your-domain.com *.your-domain.com;
```

This ensures NGINX accepts requests for your domain.

### 4. Update CORS Origins (if needed)

**File:** `manifests/demo/sortinghat.yaml`

If you need CORS support for your domain, update `SORTINGHAT_CORS_ALLOWED_ORIGINS`:

```yaml
- name: SORTINGHAT_CORS_ALLOWED_ORIGINS
  value: https://your-domain.com,https://*.your-domain.com,https://localhost:443,https://127.0.0.1:443
```

### 5. Azure DNS Integration (Optional)

**File:** `manifests/demo/nginx.yaml`

If using Azure DNS, you can add annotations to the LoadBalancer Service for automatic DNS management:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grimoirelab-gateway
  namespace: demo
  annotations:
    service.beta.kubernetes.io/azure-dns-label-name: grimoirelab-demo
    # This creates: grimoirelab-demo.<region>.cloudapp.azure.com
spec:
  type: LoadBalancer
  # ... rest of spec
```

Alternatively, manually create an A record pointing your domain to the LoadBalancer IP.

### 6. Update Documentation

**File:** `manifests/demo/README.md`

Update the Access section URLs to reflect your domain:

```markdown
- **SortingHat UI** – `https://your-domain.com/identities/` or `http://your-domain.com/identities/`
- **OpenSearch Dashboards** – `https://your-domain.com/`
```

## Deployment Steps

After making the changes:

1. **Update the TLS secret:**
   ```bash
   kubectl apply -f manifests/demo/gateway-tls-secret.yaml
   ```

2. **Apply all manifests:**
   ```bash
   kubectl apply -k manifests/demo
   ```

3. **Restart affected pods:**
   ```bash
   kubectl delete pod -n demo -l app.kubernetes.io/component=nginx
   kubectl delete pod -n demo -l app.kubernetes.io/component=sortinghat
   ```

4. **Verify DNS resolution:**
   ```bash
   nslookup your-domain.com
   # Should resolve to your LoadBalancer IP
   ```

5. **Test HTTPS access:**
   ```bash
   curl -k https://your-domain.com/identities/
   # Should return HTTP 200 or 302 (redirect)
   ```

## Verification Checklist

- [ ] TLS certificate includes domain in SANs
- [ ] SortingHat `ALLOWED_HOSTS` includes domain
- [ ] NGINX `server_name` includes domain
- [ ] DNS A record points to LoadBalancer IP
- [ ] HTTPS endpoint accessible (browser may warn about self-signed cert)
- [ ] HTTP endpoint accessible
- [ ] SortingHat UI loads at `/identities/`
- [ ] OpenSearch Dashboards loads at `/`

## Troubleshooting

**Issue:** Browser shows "SSL certificate error"
- **Solution:** The certificate must include your domain in SANs. Regenerate with the correct domain.

**Issue:** SortingHat returns "400 Bad Request"
- **Solution:** Ensure `SORTINGHAT_ALLOWED_HOST` includes your domain and restart SortingHat pods.

**Issue:** NGINX returns "404 Not Found" for domain
- **Solution:** Update `server_name` in the NGINX template and restart the gateway pod.

**Issue:** DNS not resolving
- **Solution:** Verify the A record exists and points to the correct LoadBalancer IP. Check with `kubectl get svc -n demo grimoirelab-gateway`.

## Notes

- Keep the Kubernetes service names (`grimoirelab-gateway.demo.svc`, etc.) in the certificate SANs for internal cluster communication.
- The self-signed certificate will still trigger browser warnings unless you use a certificate from a trusted CA (e.g., Let's Encrypt via cert-manager).
- Internal services (Mordred, SortingHat worker) continue to use HTTP on port 80; only external access uses HTTPS.

