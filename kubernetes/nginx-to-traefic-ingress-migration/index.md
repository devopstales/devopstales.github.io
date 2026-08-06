---
title: Migrating from NGINX Ingress Controller to Traefik: A Step-by-Step Guide
url: https://devopstales.github.io/kubernetes/nginx-to-traefic-ingress-migration/
date: 2026-02-23
keywords: kubernetes ingress migration, traefik proxy, nginx ingress retirement, ingress controller
---


The Kubernetes community announced the retirement of Ingress NGINX with best-effort maintenance ending March 2026. This guide walks you through a safe, tested migration path to Traefik Proxy—with code examples, annotation mappings, and production tips.

<!--more-->

> ⚠️ **Important**: The Kubernetes SIG Network and Security Response Committee announced that [Ingress NGINX will be retired in March 2026](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/). After this date, there will be no security patches, bug fixes, or new releases. Existing deployments will continue to function, but running unmaintained infrastructure in production carries significant risk.

If you're using `ingress-nginx` today, now is the time to plan your migration. In this post, I'll walk you through migrating to **Traefik Proxy**—a cloud-native, dynamic ingress controller with automatic service discovery and a clear migration path.

🔗 **Sources**:
- [Kubernetes Blog: Ingress NGINX Retirement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [Traefik: Migrate from NGINX to Traefik](https://doc.traefik.io/traefik/migrate/nginx-to-traefik/)

---

## Why Migrate? The Ingress NGINX Retirement Timeline

| Date | Milestone |
|------|-----------|
| **Nov 2025** | Retirement announcement by Kubernetes SIG Network & SRC |
| **Mar 2026** | Best-effort maintenance ends; no more releases or security fixes |
| **Post-Mar 2026** | Repositories become read-only; artifacts remain available |

> ✅ **Good news**: Your existing `ingress-nginx` deployments won't break. But without security updates, you'll accumulate technical debt and exposure to newly discovered vulnerabilities.

### Why Traefik?

Traefik isn't just a replacement—it's a **modernization**:

- 🚀 **Automatic service discovery**: Watches Kubernetes API; no manual config reloads
- 🔐 **Let's Encrypt built-in**: Automatic TLS provisioning and renewal
- 🧩 **Middleware architecture**: Chainable components for auth, rate limiting, headers, redirects
- 📊 **Native metrics**: Prometheus, Datadog, OpenTelemetry—no exporters needed
- 🌐 **Full protocol support**: HTTP/1.1, HTTP/2, HTTP/3, gRPC, TCP, WebSocket
- 🆓 **100% open source** (Apache 2.0), with optional enterprise features

---

## Migration Strategy: 4 Phases to Zero Downtime

```
┌─────────────────┐    ┌──────────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│ Phase 1: Assess │───>│ Phase 2: Parallel    │───>│ Phase 3: Traffic │───>│ Phase 4: Decommission│
│                 │    │ Deploy               │    │ Shift            │    │                     │
└─────────────────┘    └──────────────────────┘    └──────────────────┘    └─────────────────────┘
```

---

## Phase 1: Assess Your Current Setup

First, confirm you're using Ingress NGINX:

```bash
kubectl get pods --all-namespaces \
  --selector app.kubernetes.io/name=ingress-nginx
```

Then inventory your annotations:

```bash
kubectl get ingress -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations}{"\n"}{end}' \
  | grep nginx.ingress.kubernetes.io
```

## Phase 2: Deploy Traefik in Parallel

```bash
# Add the Traefik Helm repo
helm repo add traefik https://traefik.github.io/charts
helm repo update

# Install in a dedicated namespace
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set ingressClass.enabled=true \
  --set ingressClass.isDefaultClass=false \
  --set ports.web.redirectTo.port=websecure \
  --set ports.websecure.tls.enabled=true \
  --set providers.kubernetesIngress.enabled=true \
  --set providers.kubernetesCRD.enabled=true \
  --set replicaCount=2 \
  --set logs.level=INFO
```

Verify Installation

```bash
# Check pods and service
kubectl get pods -n traefik
kubectl get svc -n traefik

# Confirm the IngressClass exists
kubectl get ingressclass traefik -o yaml
```

## Phase 3: Migrate Ingress Resources (With Examples)

**Before (NGINX):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
      - my-app.tld
    secretName: my-app-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

**After (Traefik):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    # Path rewrite via Middleware (see advanced section)
spec:
  tls:
  - hosts:
      - my-app.tld
    secretName: my-app-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### The Zero-Modification Approach: Traefik's NGINX Provider

Here's the best part: **you don't need to modify existing Ingress resources immediately**. Traefik v3.6.2+ includes an **Ingress NGINX Provider** that understands NGINX annotations and translates them automatically.

```bash
# Install Traefik with NGINX compatibility layer
helm upgrade --install traefik traefik/traefik \
  --namespace traefik --create-namespace \
  --set providers.kubernetesIngressNginx.enabled=true
```

Or using a values file:

```yaml
# traefik-values.yaml
providers:
  kubernetesIngressNginx:
    enabled: true
```

```bash
helm upgrade --install traefik traefik/traefik \
  --namespace traefik --create-namespace \
  --values traefik-values.yaml
```

Your existing Ingress resources work **unchanged**:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"
spec:
  ingressClassName: nginx  # Traefik watches this class
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
```

---

## Annotation Migration Reference

| NGINX Annotation | Traefik Equivalent | Notes |
|------------------|-------------------|-------|
| `nginx.ingress.kubernetes.io/ssl-redirect: "true"` | `traefik.ingress.kubernetes.io/router.entrypoints: websecure` | Or use NGINX provider |
| `nginx.ingress.kubernetes.io/rewrite-target: /path` | Middleware `StripPrefix` | Requires Middleware CRD |
| `nginx.ingress.kubernetes.io/enable-cors: "true"` | Middleware `Headers` + CORS | Chainable middleware |
| `nginx.ingress.kubernetes.io/cors-allow-origin: "*"` | Middleware `Headers` | See CORS middleware docs |
| `nginx.ingress.kubernetes.io/proxy-body-size: "10m"` | `serversTransport.maxBodySize` | Global setting |
| `nginx.ingress.kubernetes.io/rate-limit: "100"` | Middleware `RateLimit` | Token bucket algorithm |
| `nginx.ingress.kubernetes.io/auth-url` | Middleware `ForwardAuth` | Direct equivalent |
| `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"` | Service `scheme: https` | In ServiceSpec |

---


## Phase 4: Traffic Migration Strategies

### Option A: DNS-Based Migration (Simple)

1. **Add Traefik to DNS** - Add Traefik's LoadBalancer IP to your DNS records alongside NGINX (round-robin)
2. **Monitor traffic** - Watch logs on both controllers to verify traffic distribution
3. **Remove NGINX from DNS** - Once validated, update DNS to point only to Traefik
4. **Wait for propagation** - Allow 24-48 hours for DNS caches to expire
5. **Uninstall NGINX** - Safe to remove after propagation window

### Option B: Weighted Traffic Shift (Production-Ready)

Use an external load balancer (Cloudflare, AWS ALB, HAProxy) in front of both controllers:

```
                    ┌─────────────────────────────────────────┐
                    │         External LB                     │
                    │  Cloudflare / AWS ALB / HAProxy         │
                    └───────────────────┬─────────────────────┘
                                        │
                          ┌─────────────┴─────────────┐
                          │                           │
                    10%   │                           │   90%
                          │                           │
                          ▼                           ▼
              ┌──────────────────┐        ┌──────────────────┐
              │      NGINX       │        │     Traefik      │
              └──────────────────┘        └──────────────────┘
```

**Gradual weight shift:**

| Stage | NGINX | Traefik | Duration |
|-------|-------|---------|----------|
| 1 | 100% | 0% | Baseline |
| 2 | 90% | 10% | 24 hours |
| 3 | 50% | 50% | 24 hours |
| 4 | 10% | 90% | 24 hours |
| 5 | 0% | 100% | Final |

---

## Phase 5: Decommission NGINX

### Step 1: Preserve the IngressClass

Before uninstalling, preserve the `nginx` IngressClass to avoid breaking existing resources:

```bash
helm upgrade ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --reuse-values \
  --set-json 'controller.ingressClassResource.annotations={"helm.sh/resource-policy": "keep"}'
```

### Step 2: Delete Admission Webhooks

```bash
kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission
kubectl delete -A MutatingWebhookConfiguration ingress-nginx-admission
```

### Step 3: Uninstall NGINX

```bash
helm uninstall ingress-nginx --namespace ingress-nginx
kubectl delete namespace ingress-nginx
```

### Step 4: Transfer LoadBalancer IP (Optional)

If you need to preserve your external IP, configure Traefik to claim it:

```yaml
# traefik-values.yaml (AWS NLB example)
service:
  type: LoadBalancer
  loadBalancerClass: service.k8s.aws/nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-eip-allocations: "eipalloc-xxx,eipalloc-yyy"
```

## Advanced: Traefik Middleware Examples

### Rate Limiting

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: default
spec:
  rateLimit:
    average: 100
    burst: 50
```

### CORS Headers

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: cors-headers
  namespace: default
spec:
  headers:
    accessControlAllowMethods:
      - GET
      - POST
      - OPTIONS
    accessControlAllowHeaders:
      - "*"
    accessControlAllowOriginList:
      - "https://example.com"
    accessControlMaxAge: 100
```

### Path Rewrite (replacement for `rewrite-target`)

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: strip-app-prefix
  namespace: default
spec:
  stripPrefix:
    prefixes:
      - /app
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: default-strip-app-prefix@kubernetescrd
spec:
  # ... rest of ingress
```

---

## Verification & Monitoring

### Check Both Controllers

```bash
kubectl get pods -n ingress-nginx
kubectl get pods -n traefik
```

### Get LoadBalancer IPs

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
kubectl get svc -n traefik traefik
```

### Test Traffic via Traefik

```bash
# Get Traefik's external IP
TRAEFIK_IP=$(kubectl get svc -n traefik traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Test with curl (bypass DNS)
curl --connect-to myapp.example.com:443:${TRAEFIK_IP} https://myapp.example.com
```

### Check Traefik Logs for Ingress Discovery

```bash
kubectl logs -n traefik deployment/traefik | grep -i "ingress"
```

### Backup Before Migration

```bash
# Export all Ingress resources
kubectl get ingress --all-namespaces -o yaml > ingress-backup.yaml

# Export NGINX ConfigMaps
kubectl get configmap --all-namespaces \
  -l app.kubernetes.io/name=ingress-nginx -o yaml > nginx-configmaps.yaml
```

---

## Key Differences: NGINX vs Traefik

| Aspect | NGINX Ingress | Traefik |
|--------|---------------|---------|
| **Configuration** | ConfigMaps, annotations | Helm values, CRDs, Gateway API |
| **Service Discovery** | Requires reload | Automatic (watches Kubernetes API) |
| **TLS/Let's Encrypt** | HTTP challenge (needs public exposure) | DNS challenge supported |
| **Metrics** | Requires prometheus-exporter sidecar | Built-in Prometheus, Datadog, OTLP |
| **Dashboard** | External addon required | Built-in web UI (`--api.dashboard=true`) |
| **TCP/UDP Routing** | Limited (ConfigMap-based) | Native via IngressRouteTCP/UDP CRDs |
| **Hot Reloading** | No (requires pod restart) | Yes (zero-downtime config updates) |

---

## Troubleshooting

### Traefik Not Picking Up Ingress

```bash
# Verify IngressClass annotation
kubectl get ingress <name> -o jsonpath='{.spec.ingressClassName}'

# Check Traefik logs
kubectl logs -n traefik deployment/traefik --tail=100

# Verify RBAC
kubectl get clusterrole traefik -o yaml | grep -A5 ingress
```

### TLS Certificate Issues

```bash
# Check certificate secret exists
kubectl get secret <tls-secret> -o yaml

# Verify Traefik TLSStore
kubectl get tlsstore -A
```

### 502 Bad Gateway

```bash
# Verify backend service endpoints
kubectl get endpoints <service-name>

# Check Traefik logs for upstream errors
kubectl logs -n traefik deployment/traefik | grep -i "upstream\|502"
```

---

## Conclusion

The retirement of Ingress NGINX is a significant moment for the Kubernetes community, but it's also an opportunity to modernize your ingress layer. Traefik offers:

✅ **Zero-downtime migration** with the NGINX compatibility provider  
✅ **No immediate code changes** to existing Ingress resources  
✅ **Modern architecture** with automatic service discovery  
✅ **Built-in features** that previously required add-ons  

**Migration checklist:**

- [ ] Inventory existing Ingress resources and annotations
- [ ] Deploy Traefik with NGINX provider enabled
- [ ] Test traffic routing through Traefik
- [ ] Implement gradual traffic shift (DNS or external LB)
- [ ] Monitor for errors during migration window
- [ ] Decommission NGINX after validation
- [ ] (Optional) Migrate to native Traefik CRDs for advanced features

Start your migration today—don't wait until March 2026 when security updates end.

---

## References

- [Kubernetes Blog: Ingress NGINX Retirement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [Traefik: Migrate from NGINX to Traefik](https://doc.traefik.io/traefik/migrate/nginx-to-traefik/)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Traefik Helm Chart](https://github.com/traefik/traefik-helm-chart)

