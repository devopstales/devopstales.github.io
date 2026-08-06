---
title: Kubernetes Egress Gateway with Custom Envoy Proxy
url: https://devopstales.github.io/kubernetes/custom-envoy-egress-proxy/
date: 2026-04-12
keywords: Envoy egress proxy, Kubernetes egress gateway, custom egress proxy, Envoy proxy, Kubernetes security, egress control, traffic management
---


Deploying a custom Envoy proxy as an egress gateway provides maximum flexibility and control over outbound traffic. This post explores building a self-hosted egress solution using Envoy's advanced features without the complexity of a full service mesh.

<!--more-->

{{< content "/filedir/egress-gateway-series.html" >}}

## Why Custom Envoy Proxy?

While service meshes like Istio provide egress gateway functionality, they come with significant complexity. A **standalone Envoy proxy** gives you:

- ✅ **Full control** over configuration
- ✅ **Lower resource usage** (no sidecars required)
- ✅ **Advanced L7 features** (routing, rate limiting, auth)
- ✅ **No vendor lock-in** (pure Envoy configuration)
- ✅ **Easier troubleshooting** (single deployment to debug)

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                           │
│                                                                 │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐                 │
│   │  Pod A   │    │  Pod B   │    │  Pod C   │                 │
│   └────┬─────┘    └────┬─────┘    └────┬─────┘                 │
│        │               │               │                        │
│        └───────────────┼───────────────┘                        │
│                        │                                        │
│                        ▼                                        │
│            ┌───────────────────────┐                            │
│            │  Envoy Egress Proxy   │ <─── Envoy Config          │
│            └───────────┬───────────┘                            │
│                        │                                        │
│           ┌────────────┴────────────┐                           │
│           │                         │                           │
│           ▼                         ▼                           │
│   ┌──────────────────┐    ┌──────────────────┐                 │
│   │  External APIs   │    │    Internet      │                 │
│   └──────────────────┘    └──────────────────┘                 │
│                                                                 │
│   ┌──────────────┐   ┌──────────────┐                          │
│   │ Prometheus   │   │    Jaeger    │                          │
│   └──────────────┘   └──────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Traffic Flow

1. **Pods configured** with egress proxy via NetworkPolicy or pod configuration
2. **Outbound traffic** routed to Envoy egress proxy deployment
3. **Envoy applies** routing rules, rate limits, authentication
4. **Traffic forwarded** to external destinations
5. **Metrics exported** to Prometheus, traces to Jaeger

## Prerequisites

| Component | Version | Notes |
|-----------|---------|-------|
| **Kubernetes** | 1.25+ | Any distribution |
| **Envoy** | 1.28+ | Latest stable |
| **kubectl** | Latest | Configured with cluster access |
| **Helm** | 3.x | Optional, for deployment |

## Installation

### Step 1: Create Egress Proxy Namespace

```bash
kubectl create namespace egress-proxy
```

### Step 2: Create Envoy Configuration

Create `envoy-egress-config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-egress-config
  namespace: egress-proxy
data:
  envoy.yaml: |
    admin:
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 9901
    
    static_resources:
      listeners:
        - name: egress_listener
          address:
            socket_address:
              address: 0.0.0.0
              port_value: 10000
          filter_chains:
            - filters:
                - name: envoy.filters.network.http_connection_manager
                  typed_config:
                    "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                    stat_prefix: egress_http
                    route_config:
                      name: egress_routes
                      virtual_hosts:
                        - name: external_services
                          domains: ["*"]
                          routes:
                            # Allow specific external APIs
                            - match:
                                prefix: "/api/"
                              route:
                                cluster: external_api_cluster
                            # Default route
                            - match:
                                prefix: "/"
                              route:
                                cluster: internet_cluster
                    http_filters:
                      - name: envoy.filters.http.router
                        typed_config:
                          "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
      
      clusters:
        - name: external_api_cluster
          connect_timeout: 30s
          type: STRICT_DNS
          lb_policy: ROUND_ROBIN
          load_assignment:
            cluster_name: external_api_cluster
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: api.external-service.com
                          port_value: 443
          transport_socket:
            name: envoy.transport_sockets.tls
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
              sni: api.external-service.com
        
        - name: internet_cluster
          connect_timeout: 30s
          type: ORIGINAL_DST
          lb_policy: CLUSTER_PROVIDED
          cleanup_interval: 30s
```

### Step 3: Deploy Envoy Egress Proxy

Create `envoy-egress-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: envoy-egress-proxy
  namespace: egress-proxy
  labels:
    app: envoy-egress-proxy
spec:
  replicas: 2  # High availability
  selector:
    matchLabels:
      app: envoy-egress-proxy
  template:
    metadata:
      labels:
        app: envoy-egress-proxy
    spec:
      containers:
        - name: envoy
          image: envoyproxy/envoy:v1.28-latest
          ports:
            - containerPort: 10000
              name: egress
            - containerPort: 9901
              name: admin
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy
              readOnly: true
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /ready
              port: 9901
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 9901
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: envoy-config
          configMap:
            name: envoy-egress-config
```

### Step 4: Create Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: envoy-egress-proxy
  namespace: egress-proxy
spec:
  selector:
    app: envoy-egress-proxy
  ports:
    - name: egress
      port: 10000
      targetPort: 10000
      protocol: TCP
    - name: admin
      port: 9901
      targetPort: 9901
      protocol: TCP
  type: ClusterIP
```

### Step 5: Apply Configuration

```bash
kubectl apply -f envoy-egress-config.yaml
kubectl apply -f envoy-egress-deployment.yaml
kubectl apply -f envoy-egress-service.yaml
```

### Step 6: Verify Deployment

```bash
# Check pods are running
kubectl get pods -n egress-proxy

# Expected output:
# NAME                                  READY   STATUS
# envoy-egress-proxy-xxxxxxxxxx-xxxxx   1/1     Running
# envoy-egress-proxy-yyyyyyyyyy-yyyyy   1/1     Running

# Check service
kubectl get svc -n egress-proxy

# Test connectivity
kubectl run test-pod --image=curlimages/curl -it --rm --restart=Never \
  --namespace egress-proxy \
  -- curl -v http://envoy-egress-proxy.egress-proxy.svc:10000/api/
```

## Advanced Configuration

### Rate Limiting

```yaml
# Add to envoy.yaml static_resources.listeners[0].filter_chains[0].filters[0]
http_filters:
  - name: envoy.filters.http.local_ratelimit
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
      stat_prefix: egress_rate_limit
      token_bucket:
        max_tokens: 1000
        tokens_per_fill: 1000
        fill_interval: 60s
      filter_enabled:
        runtime_key: local_rate_limit_enabled
        default_value:
          numerator: 100
          denominator: HUNDRED
      filter_enforced:
        runtime_key: local_rate_limit_enforced
        default_value:
          numerator: 100
          denominator: HUNDRED
  - name: envoy.filters.http.router
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

### mTLS to External Services

```yaml
# Add to clusters section
clusters:
  - name: secure_api_cluster
    connect_timeout: 30s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: secure_api_cluster
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: secure-api.example.com
                    port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: secure-api.example.com
        common_tls_context:
          tls_certificates:
            - certificate_chain:
                filename: /etc/envoy/certs/client.crt
              private_key:
                filename: /etc/envoy/certs/client.key
          validation_context:
            trusted_ca:
              filename: /etc/envoy/certs/ca.crt
```

### Circuit Breaker

```yaml
# Add to clusters section
clusters:
  - name: external_api_cluster
    connect_timeout: 30s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    circuit_breakers:
      thresholds:
        - priority: DEFAULT
          max_connections: 100
          max_pending_requests: 100
          max_requests: 1000
          max_retries: 3
    load_assignment:
      # ... rest of configuration
```

### Retry Policy

```yaml
# Add to route configuration
routes:
  - match:
      prefix: "/api/"
    route:
      cluster: external_api_cluster
      retry_policy:
        retry_on: "5xx,reset,connect-failure,retriable-4xx"
        num_retries: 3
        per_try_timeout: 2s
        back_off:
          base_interval: 0.25s
          max_interval: 30s
```

## Routing Traffic to Egress Proxy

### Method 1: NetworkPolicy (CNI-dependent)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: force-egress-proxy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    
    # Force all other traffic through egress proxy
    - to:
        - namespaceSelector:
            matchLabels:
              name: egress-proxy
          podSelector:
            matchLabels:
              app: envoy-egress-proxy
      ports:
        - protocol: TCP
          port: 10000
```

### Method 2: Pod Configuration (HTTP_PROXY)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: myapp/api:v1.0
          env:
            - name: HTTP_PROXY
              value: "http://envoy-egress-proxy.egress-proxy.svc:10000"
            - name: HTTPS_PROXY
              value: "http://envoy-egress-proxy.egress-proxy.svc:10000"
            - name: NO_PROXY
              value: "localhost,.svc.cluster.local,.cluster.local"
```

### Method 3: Service Mesh Integration

If using a service mesh, configure egress gateway to route to Envoy:

```yaml
# Istio ServiceEntry + VirtualService
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: egress-proxy
  namespace: istio-system
spec:
  hosts:
    - "egress-proxy.egress-proxy.svc.cluster.local"
  ports:
    - number: 10000
      name: http
      protocol: HTTP
  location: MESH_INTERNAL
  resolution: DNS

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: route-to-egress-proxy
  namespace: production
spec:
  hosts:
    - "api.external-service.com"
  http:
    - route:
        - destination:
            host: egress-proxy.egress-proxy.svc.cluster.local
            port:
              number: 10000
```

## Monitoring and Observability

### Prometheus Metrics

```yaml
# Add to envoy.yaml
stats_sinks:
  - name: envoy.stat_sinks.statsd
    typed_config:
      "@type": type.googleapis.com/envoy.config.metrics.v3.StatsdSink
      tcp_cluster_name: statsd_cluster

# Or use Prometheus scraper
clusters:
  - name: prometheus_stats
    connect_timeout: 0.25s
    type: STATIC
    load_assignment:
      cluster_name: prometheus_stats
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 127.0.0.1
                    port_value: 9901
```

### Access Logging

```yaml
# Add to HttpConnectionManager
access_log:
  - name: envoy.access_loggers.file
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
      path: /dev/stdout
      log_format:
        text_format: "[%START_TIME%] %REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL% %RESPONSE_CODE% %RESPONSE_FLAGS% %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %REQ(X-FORWARDED-FOR)% %REQ(USER-AGENT)% %REQ(X-REQUEST-ID)% %REQ(:AUTHORITY)% %UPSTREAM_HOST%\n"
```

### Distributed Tracing

```yaml
# Add to HttpConnectionManager
tracing:
  provider:
    name: envoy.tracers.zipkin
    typed_config:
      "@type": type.googleapis.com/envoy.config.trace.v3.ZipkinConfig
      collector_cluster: zipkin
      collector_endpoint: "/api/v2/spans"
      collector_endpoint_version: HTTP_JSON
      shared_span_context: false

# Add cluster for Zipkin
clusters:
  - name: zipkin
    connect_timeout: 1s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: zipkin
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: zipkin.observability.svc
                    port_value: 9411
```

### Grafana Dashboard

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-egress-dashboard
  namespace: monitoring
data:
  envoy-egress.json: |
    {
      "dashboard": {
        "title": "Envoy Egress Proxy",
        "panels": [
          {
            "title": "Request Rate",
            "targets": [
              {
                "expr": "rate(envoy_cluster_upstream_rq_total[5m])"
              }
            ]
          },
          {
            "title": "Error Rate",
            "targets": [
              {
                "expr": "rate(envoy_cluster_upstream_rq_5xx[5m])"
              }
            ]
          },
          {
            "title": "Latency (p99)",
            "targets": [
              {
                "expr": "histogram_quantile(0.99, rate(envoy_cluster_upstream_rq_time_bucket[5m]))"
              }
            ]
          },
          {
            "title": "Active Connections",
            "targets": [
              {
                "expr": "envoy_cluster_upstream_cx_active"
              }
            ]
          }
        ]
      }
    }
```

## Security Configuration

### JWT Validation

```yaml
# Add to http_filters
http_filters:
  - name: envoy.filters.http.jwt_authn
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
      providers:
        - name: auth0
          issuer: "https://auth.example.com"
          audiences:
            - "egress-api"
          remote_jwks:
            http_uri:
              uri: "https://auth.example.com/.well-known/jwks.json"
              cluster: auth0_jwks
              timeout: 5s
            cache_duration:
              seconds: 600
      rules:
        - match:
            prefix: "/"
          requires:
            provider_name: auth0
  - name: envoy.filters.http.router
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

# Add JWKS cluster
clusters:
  - name: auth0_jwks
    connect_timeout: 5s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: auth0_jwks
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: auth.example.com
                    port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: auth.example.com
```

### IP Whitelisting

```yaml
# Add to route configuration
routes:
  - match:
      prefix: "/api/"
    route:
      cluster: external_api_cluster
    request_headers_to_add:
      - header:
          key: "X-Forwarded-For"
          value: "%DOWNSTREAM_REMOTE_ADDRESS%"
    typed_per_filter_config:
      envoy.filters.http.ip_tagging:
        "@type": type.googleapis.com/envoy.extensions.filters.http.ip_tagging.v3.IPTagging
        ip_tagging_type: BOTH
        ip_tags:
          - ip_tag_name: "allowed_networks"
            ip_list:
              - address_prefix: "10.0.0.0/8"
              - address_prefix: "172.16.0.0/12"
              - address_prefix: "192.168.0.0/16"
```

## Troubleshooting

### Issue: Pods Can't Reach External Services

```bash
# Check Envoy pods are running
kubectl get pods -n egress-proxy

# Verify Envoy config
kubectl exec -n egress-proxy envoy-egress-proxy-xxxxx -- \
  curl -s localhost:9901/config_dump

# Test from inside proxy
kubectl exec -n egress-proxy envoy-egress-proxy-xxxxx -it -- \
  curl -v http://localhost:10000/api/

# Check pod proxy configuration
kubectl get pod api-server-xxxxx -n production -o yaml | grep -i proxy
```

### Issue: High Latency Through Proxy

```bash
# Check Envoy stats
kubectl exec -n egress-proxy envoy-egress-proxy-xxxxx -- \
  curl -s localhost:9901/stats | grep upstream_rq_time

# Check circuit breaker status
kubectl exec -n egress-proxy envoy-egress-proxy-xxxxx -- \
  curl -s localhost:9901/stats | grep circuit_breaker

# Verify connection pool
kubectl exec -n egress-proxy envoy-egress-proxy-xxxxx -- \
  curl -s localhost:9901/stats | grep cx_active
```

### Issue: Rate Limiting Not Working

```bash
# Check rate limit stats
kubectl exec -n egress-proxy envoy-egress-proxy-xxxxx -- \
  curl -s localhost:9901/stats | grep rate_limit

# Verify config has rate limit filter
kubectl get configmap envoy-egress-config -n egress-proxy -o yaml | \
  grep -A20 local_ratelimit
```

### Common Problems and Solutions

| Problem | Cause | Solution |
|---------|-------|----------|
| Connection refused | Envoy not listening | Check port configuration |
| 503 Service Unavailable | Upstream cluster down | Verify external service |
| Rate limit not enforced | Filter not in chain | Check filter order |
| mTLS handshake failed | Certificate issue | Verify cert paths |
| High latency | Connection pool exhausted | Increase circuit breaker limits |

## Comparison with Other Solutions

| Feature | Custom Envoy | Istio | Cilium | Monzo Operator |
|---------|-------------|-------|--------|----------------|
| **Complexity** | Medium | High | Medium | Low |
| **Resource Usage** | Low | High (sidecars) | Low | N/A (AWS) |
| **L7 Features** | Full | Full | Limited | Limited |
| **mTLS** | Manual config | Automatic | Manual | AWS managed |
| **Observability** | Manual setup | Built-in | Hubble | CloudWatch |
| **Cloud Provider** | Any | Any | Any | AWS only |
| **Operations** | Self-managed | Self-managed | Self-managed | AWS managed |

### When to Choose Custom Envoy

**Choose Custom Envoy when:**
- ✅ Need full control over egress configuration
- ✅ Want L7 features without service mesh complexity
- ✅ Resource efficiency is important
- ✅ Multi-cloud deployment
- ✅ Team has Envoy expertise

**Consider alternatives when:**
- 📋 Want automatic mTLS (choose Istio)
- 📋 Need eBPF performance (choose Cilium)
- 📋 Running only on AWS (choose Monzo Operator)
- 📋 Want zero operations (choose cloud NAT)

## Next Steps

In the next post of this series:

- **Squid Proxy on Kubernetes** - Traditional HTTP proxy approach
- Comparison with Envoy
- Use cases for each solution

## Conclusion

Custom Envoy Proxy provides:

**Advantages:**
- ✅ Full control over egress configuration
- ✅ Advanced L7 features (routing, rate limiting, auth)
- ✅ Lower resource usage than service mesh
- ✅ No vendor lock-in
- ✅ Multi-cloud compatible
- ✅ Rich observability (metrics, logs, traces)

**Considerations:**
- 📋 Self-managed (operations overhead)
- 📋 Requires Envoy expertise
- 📋 Manual mTLS configuration
- 📋 No automatic sidecar injection

For organizations needing **maximum flexibility** with advanced traffic management features without service mesh complexity, a custom Envoy egress proxy is an excellent choice.

