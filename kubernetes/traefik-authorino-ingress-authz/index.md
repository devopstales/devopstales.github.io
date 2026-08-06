---
title: Zero-Trust Ingress with Traefik + Authorino - Group-Based Authorization
url: https://devopstales.github.io/kubernetes/traefik-authorino-ingress-authz/
date: 2026-04-22
keywords: Traefik authorization, Authorino Kubernetes, ingress-based AuthZ, group-based access, zero-trust ingress, OIDC authorization, Kubernetes security
---


Implementing group-based authorization at the ingress level ensures that sensitive applications are only accessible to authorized user groups. This post covers implementing zero-trust ingress with **Traefik + Authorino** where different ingress routes are only visible to specific user groups.

<!--more-->

{{< content "/filedir/egress-gateway-series.html" >}}

## Architecture Overview

```bash
┌─────────────────────────────────────────────────────────────────┐
│                        User Request                             │
│                    (with OIDC Token)                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Traefik Ingress                              │
│  ┌─────────────────┐         ┌─────────────────┐               │
│  │  Ingress 1      │         │  Ingress 2      │               │
│  │  (admin-app)    │         │  (dev-app)      │               │
│  │                 │         │                 │               │
│  │  Group: admin   │         │  Group: dev     │               │
│  └────────┬────────┘         └────────┬────────┘               │
└───────────┼───────────────────────────┼────────────────────────┘
            │                           │
            ▼                           ▼
    ┌───────────────────────────────────────────────────┐
    │              Authorino (AuthZ)                    │
    │                                                   │
    │  ┌─────────────────┐     ┌─────────────────┐     │
    │  │  Policy 1       │     │  Policy 2       │     │
    │  │  group:admin    │     │  group:dev      │     │
    │  │  → Allow        │     │  → Allow        │     │
    │  └─────────────────┘     └─────────────────┘     │
    └───────────────────────────────────────────────────┘
                         │
                         ▼
            ┌────────────────────────┐
            │   OIDC Provider        │
            │   (Keycloak/Auth0)     │
            │   - User Groups        │
            │   - JWT Validation     │
            └────────────────────────┘
```

## Why Traefik + Authorino?

| Feature | Traefik + Authorino | Alternatives |
|---------|---------------------|--------------|
| **External AuthZ** | ✅ Native support | ⚠️ Custom middleware |
| **Group-Based AuthZ** | ✅ Built-in | ⚠️ Manual implementation |
| **OIDC Integration** | ✅ Multiple providers | ✅ Similar |
| **Performance** | ✅ External service | ⚠️ Inline processing |
| **Flexibility** | ✅ Policy per route | ⚠️ Global policies |
| **Kubernetes Native** | ✅ CRD-based | ⚠️ ConfigMap-based |

## Prerequisites

| Component | Version | Notes |
|-----------|---------|-------|
| **Kubernetes** | 1.26+ | Tested on 1.28, 1.29 |
| **Traefik** | 2.10+ | With Middleware support |
| **Authorino** | 0.18+ | Authorization service |
| **OIDC Provider** | Any | Keycloak, Auth0, Okta |
| **cert-manager** | 1.13+ | For TLS certificates |

## Step 1: Install Traefik Ingress Controller

### Install Traefik via Helm

```bash
# Add Traefik Helm repository
helm repo add traefik https://traefik.github.io/charts
helm repo update

# Install Traefik with external auth middleware enabled
helm install traefik traefik/traefik \
  --namespace traefik-system \
  --create-namespace \
  --set ingressClass.enabled=true \
  --set ingressClass.isDefaultClass=true \
  --set providers.kubernetesIngress.enabled=true \
  --set providers.kubernetesCRD.enabled=true \
  --set providers.kubernetesGateway.enabled=true \
  --set ports.web.redirectTo.port=websecure \
  --set ports.websecure.http3.enabled=true \
  --wait
```

### Verify Traefik Installation

```bash
# Check Traefik pods
kubectl get pods -n traefik-system

# Expected output:
# NAME                      READY   STATUS
# traefik-xxxxxxxxxx-xxxxx  1/1     Running

# Get Traefik dashboard URL
kubectl get svc -n traefik-system traefik-dashboard
```

## Step 2: Install Authorino

### Install Authorino via Helm

```bash
# Add Authorino Helm repository
helm repo add authorino https://kuadrant.github.io/authorino/helm-charts
helm repo update

# Install Authorino
helm install authorino authorino/authorino \
  --namespace authorino-system \
  --create-namespace \
  --set config.authConfigLabelSelectors="security.kuadrant.io/protected-by=authorino" \
  --set listener.watchingNamespaces="*" \
  --wait
```

### Verify Authorino Installation

```bash
# Check Authorino pods
kubectl get pods -n authorino-system

# Expected output:
# NAME                        READY   STATUS
# authorino-xxxxxxxxxx-xxxxx  1/1     Running

# Check Authorino service
kubectl get svc -n authorino-system authorino-service
```

## Step 3: Configure OIDC Provider

### Keycloak Realm Setup

```bash
# Create realm (via Keycloak admin or CLI)
kcadm.sh create realms -s realm=production -s enabled=true

# Create groups
kcadm.sh create groups -r production -s name=admin
kcadm.sh create groups -r production -s name=dev

# Create users and assign to groups
kcadm.sh create users -r production -s username=admin1 -s enabled=true
kcadm.sh add-roles -r production --uusername admin1 --gname admin

kcadm.sh create users -r production -s username=dev1 -s enabled=true
kcadm.sh add-roles -r production --uusername dev1 --gname dev
```

### Create OIDC Client

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: oidc-client-secret
  namespace: default
type: Opaque
stringData:
  client-id: "traefik-ingress"
  client-secret: "your-client-secret-here"
  issuer: "https://keycloak.mydomain.intra/realms/production"
  jwks-url: "https://keycloak.mydomain.intra/realms/production/protocol/openid-connect/certs"
```

## Step 4: Create Authorino AuthConfig

### Admin Group Authorization Policy

```yaml
apiVersion: authorino.kuadrant.io/v1beta2
kind: AuthConfig
metadata:
  name: admin-ingress-authz
  namespace: default
  labels:
    security.kuadrant.io/protected-by: authorino
spec:
  hosts:
    - "admin.mydomain.intra"
  
  authentication:
    "oidc-jwt":
      jwt:
        issuer: "https://keycloak.mydomain.intra/realms/production"
        jwksUrl: "https://keycloak.mydomain.intra/realms/production/protocol/openid-connect/certs"
        ttl: 3600
        secretRef:
          name: oidc-client-secret
  
  authorization:
    "admin-group-check":
      patternMatching:
        patterns:
          - any:
              - selector: auth.identity.groups
                operator: incl
                value: "admin"
              - selector: auth.identity.realm_access.roles
                operator: incl
                value: "admin"
  
  response:
    successHeaders:
      "x-authenticated-user":
        selector: auth.identity.preferred_username
      "x-authenticated-groups":
        selector: auth.identity.groups
```

### Developer Group Authorization Policy

```yaml
apiVersion: authorino.kuadrant.io/v1beta2
kind: AuthConfig
metadata:
  name: dev-ingress-authz
  namespace: default
  labels:
    security.kuadrant.io/protected-by: authorino
spec:
  hosts:
    - "dev.mydomain.intra"
  
  authentication:
    "oidc-jwt":
      jwt:
        issuer: "https://keycloak.mydomain.intra/realms/production"
        jwksUrl: "https://keycloak.mydomain.intra/realms/production/protocol/openid-connect/certs"
        ttl: 3600
        secretRef:
          name: oidc-client-secret
  
  authorization:
    "dev-group-check":
      patternMatching:
        patterns:
          - any:
              - selector: auth.identity.groups
                operator: incl
                value: "dev"
              - selector: auth.identity.realm_access.roles
                operator: incl
                value: "dev"
  
  response:
    successHeaders:
      "x-authenticated-user":
        selector: auth.identity.preferred_username
      "x-authenticated-groups":
        selector: auth.identity.groups
```

### Apply AuthConfig Resources

```bash
kubectl apply -f admin-authz.yaml
kubectl apply -f dev-authz.yaml

# Verify AuthConfig status
kubectl get authconfig -n default

# Expected output:
# NAME                  READY   REASON
# admin-ingress-authz   True    AuthorinoReady
# dev-ingress-authz     True    AuthorinoReady
```

## Step 5: Create Traefik Middleware

### External Auth Middleware for Admin

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: admin-authz-middleware
  namespace: default
spec:
  forwardAuth:
    address: "http://authorino-service.authorino-system.svc.cluster.local:80"
    trustForwardHeader: true
    authResponseHeaders:
      - "x-authenticated-user"
      - "x-authenticated-groups"
    authRequestHeaders:
      - "authorization"
      - "cookie"
```

### External Auth Middleware for Dev

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: dev-authz-middleware
  namespace: default
spec:
  forwardAuth:
    address: "http://authorino-service.authorino-system.svc.cluster.local:80"
    trustForwardHeader: true
    authResponseHeaders:
      - "x-authenticated-user"
      - "x-authenticated-groups"
    authRequestHeaders:
      - "authorization"
      - "cookie"
```

### Apply Middleware Resources

```bash
kubectl apply -f admin-middleware.yaml
kubectl apply -f dev-middleware.yaml

# Verify middleware
kubectl get middleware -n default

# Expected output:
# NAME                   AGE
# admin-authz-middleware  5m
# dev-authz-middleware    5m
```

## Step 6: Create Ingress Routes

### Admin Ingress (Group: admin only)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: admin-app-ingress
  namespace: default
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: "default-admin-authz-middleware@kubernetescrd"
    traefik.ingress.kubernetes.io/router.entrypoints: "websecure"
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - "admin.mydomain.intra"
      secretName: admin-tls-secret
  rules:
    - host: "admin.mydomain.intra"
      http:
        paths:
          - path: "/"
            pathType: Prefix
            backend:
              service:
                name: admin-app
                port:
                  number: 8080
```

### Developer Ingress (Group: dev only)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dev-app-ingress
  namespace: default
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: "default-dev-authz-middleware@kubernetescrd"
    traefik.ingress.kubernetes.io/router.entrypoints: "websecure"
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - "dev.mydomain.intra"
      secretName: dev-tls-secret
  rules:
    - host: "dev.mydomain.intra"
      http:
        paths:
          - path: "/"
            pathType: Prefix
            backend:
              service:
                name: dev-app
                port:
                  number: 8080
```

### Apply Ingress Resources

```bash
kubectl apply -f admin-ingress.yaml
kubectl apply -f dev-ingress.yaml

# Verify ingress
kubectl get ingress -n default

# Expected output:
# NAME               CLASS    HOSTS                  ADDRESS        PORTS     AGE
# admin-app-ingress  traefik  admin.mydomain.intra   192.168.1.10   80, 443   5m
# dev-app-ingress    traefik  dev.mydomain.intra     192.168.1.10   80, 443   5m
```

## Step 7: Deploy Test Applications

### Admin Application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: admin-app
  template:
    metadata:
      labels:
        app: admin-app
    spec:
      containers:
        - name: admin-app
          image: nginx:alpine
          ports:
            - containerPort: 80
          env:
            - name: APP_NAME
              value: "Admin Application"
            - name: REQUIRED_GROUP
              value: "admin"
---
apiVersion: v1
kind: Service
metadata:
  name: admin-app
  namespace: default
spec:
  selector:
    app: admin-app
  ports:
    - port: 8080
      targetPort: 80
  type: ClusterIP
```

### Developer Application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dev-app
  template:
    metadata:
      labels:
        app: dev-app
    spec:
      containers:
        - name: dev-app
          image: nginx:alpine
          ports:
            - containerPort: 80
          env:
            - name: APP_NAME
              value: "Developer Application"
            - name: REQUIRED_GROUP
              value: "dev"
---
apiVersion: v1
kind: Service
metadata:
  name: dev-app
  namespace: default
spec:
  selector:
    app: dev-app
  ports:
    - port: 8080
      targetPort: 80
  type: ClusterIP
```

### Apply Test Applications

```bash
kubectl apply -f admin-app.yaml
kubectl apply -f dev-app.yaml

# Verify deployments
kubectl get deployments -n default

# Verify services
kubectl get svc -n default
```

## Testing Authorization

### Test 1: Admin User Accessing Admin App

```bash
# Get admin user token
ADMIN_TOKEN=$(curl -X POST \
  https://keycloak.mydomain.intra/realms/production/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=traefik-ingress" \
  -d "client_secret=your-client-secret" \
  -d "username=admin1" \
  -d "password=admin1password" \
  | jq -r '.access_token')

# Access admin app with admin token
curl -v https://admin.mydomain.intra \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Host: admin.mydomain.intra"

# Expected: 200 OK with x-authenticated-user header
```

### Test 2: Admin User Accessing Dev App (Should Fail)

```bash
# Access dev app with admin token
curl -v https://dev.mydomain.intra \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Host: dev.mydomain.intra"

# Expected: 403 Forbidden
# Response: {"error": "authorization failed", "reason": "group check failed"}
```

### Test 3: Dev User Accessing Dev App

```bash
# Get dev user token
DEV_TOKEN=$(curl -X POST \
  https://keycloak.mydomain.intra/realms/production/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=traefik-ingress" \
  -d "client_secret=your-client-secret" \
  -d "username=dev1" \
  -d "password=dev1password" \
  | jq -r '.access_token')

# Access dev app with dev token
curl -v https://dev.mydomain.intra \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -H "Host: dev.mydomain.intra"

# Expected: 200 OK with x-authenticated-user header
```

### Test 4: Dev User Accessing Admin App (Should Fail)

```bash
# Access admin app with dev token
curl -v https://admin.mydomain.intra \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -H "Host: admin.mydomain.intra"

# Expected: 403 Forbidden
# Response: {"error": "authorization failed", "reason": "group check failed"}
```

### Test 5: Unauthenticated Access (Should Fail)

```bash
# Access without token
curl -v https://admin.mydomain.intra \
  -H "Host: admin.mydomain.intra"

# Expected: 401 Unauthorized
# Response: {"error": "unauthenticated", "reason": "missing authorization header"}
```

## Advanced Configuration

### Multiple Groups per Ingress

```yaml
apiVersion: authorino.kuadrant.io/v1beta2
kind: AuthConfig
metadata:
  name: multi-group-authz
  namespace: default
spec:
  hosts:
    - "shared.mydomain.intra"
  
  authentication:
    "oidc-jwt":
      jwt:
        issuer: "https://keycloak.mydomain.intra/realms/production"
        jwksUrl: "https://keycloak.mydomain.intra/realms/production/protocol/openid-connect/certs"
  
  authorization:
    "multi-group-check":
      patternMatching:
        patterns:
          - any:
              - selector: auth.identity.groups
                operator: incl
                value: "admin"
              - selector: auth.identity.groups
                operator: incl
                value: "dev"
              - selector: auth.identity.groups
                operator: incl
                value: "ops"
```

### Role-Based + Group-Based Authorization

```yaml
authorization:
  "role-and-group-check":
    patternMatching:
      patterns:
        - all:
            - selector: auth.identity.groups
              operator: incl
              value: "admin"
            - any:
                - selector: auth.identity.realm_access.roles
                  operator: incl
                  value: "admin.read"
                - selector: auth.identity.realm_access.roles
                  operator: incl
                  value: "admin.write"
```

### Rate Limiting per User Group

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit-by-group
  namespace: default
spec:
  rateLimit:
    average: 100
    burst: 50
    headerField: "x-authenticated-groups"
```

### Logging and Audit

```yaml
apiVersion: authorino.kuadrant.io/v1beta2
kind: AuthConfig
metadata:
  name: audited-authz
  namespace: default
spec:
  hosts:
    - "secure.mydomain.intra"
  
  authentication:
    "oidc-jwt":
      jwt:
        issuer: "https://keycloak.mydomain.intra/realms/production"
        jwksUrl: "https://keycloak.mydomain.intra/realms/production/protocol/openid-connect/certs"
  
  authorization:
    "group-check":
      patternMatching:
        patterns:
          - selector: auth.identity.groups
            operator: incl
            value: "admin"
  
  # Audit logging
  callbacks:
    "audit-log":
      http:
        url: "http://audit-service.audit-system.svc.cluster.local:8080/log"
        method: "POST"
        body:
          selector: auth
```

## Monitoring and Observability

### Authorino Metrics

```bash
# Enable Authorino metrics
helm upgrade authorino authorino/authorino \
  --namespace authorino-system \
  --set metrics.enabled=true \
  --set metrics.port=8080

# Access metrics
kubectl port-forward -n authorino-system svc/authorino-service 8080:8080
```

### Key Metrics

| Metric | Description |
|--------|-------------|
| `authorino_auth_request_total` | Total auth requests |
| `authorino_auth_request_allowed` | Allowed requests |
| `authorino_auth_request_denied` | Denied requests |
| `authorino_auth_latency_seconds` | Authorization latency |

### Grafana Dashboard

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: authorino-dashboard
  namespace: monitoring
data:
  authorino.json: |
    {
      "dashboard": {
        "title": "Authorino Authorization",
        "panels": [
          {
            "title": "Auth Requests",
            "targets": [
              {
                "expr": "rate(authorino_auth_request_total[5m])"
              }
            ]
          },
          {
            "title": "Allowed vs Denied",
            "targets": [
              {
                "expr": "rate(authorino_auth_request_allowed[5m])"
              },
              {
                "expr": "rate(authorino_auth_request_denied[5m])"
              }
            ]
          },
          {
            "title": "Auth Latency (p99)",
            "targets": [
              {
                "expr": "histogram_quantile(0.99, rate(authorino_auth_latency_seconds_bucket[5m]))"
              }
            ]
          }
        ]
      }
    }
```

### Traefik Access Logs

```yaml
# Enable access logs in Traefik
helm upgrade traefik traefik/traefik \
  --namespace traefik-system \
  --set logs.access.enabled=true \
  --set logs.access.format=json \
  --set logs.access.filters.statuscodes=["200-299","exclude"]
```

## Troubleshooting

### Issue: 401 Unauthorized

```bash
# Check token validity
curl -X POST \
  https://keycloak.mydomain.intra/realms/production/protocol/openid-connect/token/introspect \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "token=YOUR_TOKEN" \
  -d "client_id=traefik-ingress" \
  -d "client_secret=your-client-secret"

# Check Authorino logs
kubectl logs -n authorino-system -l app.kubernetes.io/name=authorino

# Check Traefik logs
kubectl logs -n traefik-system -l app.kubernetes.io/name=traefik
```

### Issue: 403 Forbidden (Unexpected)

```bash
# Verify user groups in token
echo $TOKEN | jq -R 'split(".") | .[1] | @base64d | fromjson' | jq '.groups'

# Check AuthConfig status
kubectl get authconfig admin-ingress-authz -n default -o yaml

# Verify pattern matching
kubectl logs -n authorino-system -l app.kubernetes.io/name=authorino | grep "admin-ingress-authz"
```

### Issue: Authorino Not Responding

```bash
# Check Authorino service
kubectl get svc -n authorino-system authorino-service

# Test connectivity from Traefik
kubectl exec -n traefik-system traefik-xxxxxxxxxx -- \
  curl -v http://authorino-service.authorino-system.svc.cluster.local:80/healthz

# Check Authorino pod status
kubectl get pods -n authorino-system
kubectl describe pod -n authorino-system authorino-xxxxxxxxxx
```

### Common Problems and Solutions

| Problem | Cause | Solution |
|---------|-------|----------|
| 401 Unauthorized | Missing/invalid token | Verify OIDC configuration |
| 403 Forbidden | Group mismatch | Check user groups in Keycloak |
| 503 Service Unavailable | Authorino down | Check Authorino pod status |
| Timeout | Network issue | Verify service connectivity |
| JWKS fetch failed | Network/Cert issue | Check issuer URL accessibility |

## Security Best Practices

### 1. Token Validation

```yaml
authentication:
  "oidc-jwt":
    jwt:
      issuer: "https://keycloak.mydomain.intra/realms/production"
      jwksUrl: "https://keycloak.mydomain.intra/realms/production/protocol/openid-connect/certs"
      ttl: 3600  # Cache JWKS for 1 hour
      refreshInterval: 300  # Refresh every 5 minutes
```

### 2. Least Privilege Groups

```yaml
authorization:
  "least-privilege-check":
    patternMatching:
      patterns:
        - all:  # Require ALL conditions
            - selector: auth.identity.groups
              operator: incl
              value: "admin"
            - selector: auth.identity.realm_access.roles
              operator: incl
              value: "admin.read"
```

### 3. Audit Logging

```yaml
callbacks:
  "audit-log":
    http:
      url: "http://audit-service.audit-system.svc.cluster.local:8080/log"
      method: "POST"
      body:
        selector: auth
      headers:
        "Content-Type": "application/json"
```

### 4. Rate Limiting

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: auth-rate-limit
  namespace: default
spec:
  rateLimit:
    average: 100
    burst: 50
    sourceCriterion:
      requestHeaderName: "x-authenticated-user"
```

## Comparison with Alternatives

| Feature | Traefik+Authorino | Istio | OPA Gatekeeper |
|---------|-------------------|-------|----------------|
| **External AuthZ** | ✅ Native | ⚠️ Custom | ❌ No |
| **Group-Based** | ✅ Built-in | ✅ Yes | ⚠️ Manual |
| **OIDC Integration** | ✅ Multiple | ✅ Yes | ⚠️ Manual |
| **Performance** | ✅ External | ⚠️ Sidecar | ✅ External |
| **Complexity** | 🟡 Medium | 🔴 High | 🟡 Medium |
| **Kubernetes Native** | ✅ CRD | ✅ CRD | ✅ CRD |

## When to Use This Approach

**Choose Traefik + Authorino when:**
- ✅ Need group-based authorization at ingress
- ✅ Want external authorization service
- ✅ Using OIDC for authentication
- ✅ Need fine-grained policy control
- ✅ Want Kubernetes-native CRD approach

**Consider alternatives when:**
- 📋 Need service mesh features (choose Istio)
- 📋 Want policy-as-code (choose OPA)
- 📋 Need L7 traffic management only (choose Traefik alone)
- 📋 Running on managed Kubernetes with cloud auth (use cloud-native)

## Conclusion

Traefik + Authorino provides:

**Advantages:**
- ✅ Native external authorization support
- ✅ Group-based access control
- ✅ OIDC integration out-of-box
- ✅ Kubernetes CRD-based configuration
- ✅ Fine-grained policy per ingress
- ✅ Audit logging capabilities
- ✅ Rate limiting integration

**Considerations:**
- 📋 Additional service to manage (Authorino)
- 📋 Network latency for auth checks
- 📋 OIDC provider required
- 📋 Policy syntax learning curve

For organizations needing **zero-trust ingress with group-based authorization**, Traefik + Authorino provides a robust, Kubernetes-native solution that integrates seamlessly with existing OIDC providers.

---

*For more Kubernetes security guides, check out our [K8s Security Best Practices](/kubernetes/k8s-security/) and [RBAC Guide](/kubernetes/k8s-rbac/).*

