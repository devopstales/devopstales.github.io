---
title: Kubernetes Egress Gateway with Antrea
url: https://devopstales.github.io/kubernetes/antrea-egress-gateway/
date: 2026-04-05
keywords: Antrea egress, Kubernetes egress gateway, Antrea CNI, Open vSwitch, network policy, egress IP, NAT gateway
---


Antrea is an open source Kubernetes CNI based on Open vSwitch that provides advanced networking features including **Egress Gateway** - completely free and open source. This post explores Antrea's egress capabilities as a true open source alternative.

<!--more-->

{{< content "/filedir/egress-gateway-series.html" >}}

## Why Antrea for Egress?

Unlike Calico (where egress gateway requires Enterprise), **Antrea includes full egress gateway functionality in the open source version**.

### Antrea vs Calico for Egress

| Feature | Antrea (Open Source) | Calico (Open Source) | Calico Enterprise |
|---------|---------------------|---------------------|-------------------|
| **Egress Gateway** | ✅ Included | ❌ Not available | ✅ Available |
| **ExternalNode** | ✅ Supported | ❌ Not available | ✅ Available |
| **IP Pool Management** | ✅ Included | ⚠️ Limited | ✅ Full |
| **Network Policy** | ✅ Kubernetes NP + Extensions | ✅ Kubernetes NP | ✅ Enhanced NP |
| **Encryption** | ✅ IPsec/WireGuard | ⚠️ WireGuard only | ✅ WireGuard |
| **Cost** | Free | Free | Paid |

### Antrea Architecture

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
│            │  OVS Bridge br-int    │                            │
│            └───────────┬───────────┘                            │
│                        │                                        │
│                        ▼                                        │
│            ┌───────────────────────┐                            │
│            │   Antrea Agent        │                            │
│            └───────────┬───────────┘                            │
│                        │                                        │
│                        ▼                                        │
│            ┌───────────────────────┐                            │
│            │   Egress IP Pool      │                            │
│            └───────────┬───────────┘                            │
│                        │                                        │
│                        ▼                                        │
│            ┌───────────────────────┐                            │
│            │   Gateway Node        │                            │
│            └───────────┬───────────┘                            │
│                        │                                        │
│                        ▼                                        │
│            ┌───────────────────────┐                            │
│            │  External Network     │                            │
│            └───────────────────────┘                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Purpose |
|-----------|---------|
| **Antrea Controller** | Centralized configuration management |
| **Antrea Agent** | Per-node networking (runs as DaemonSet) |
| **OVS Bridge** | Open vSwitch for packet forwarding |
| **Egress IP Pool** | Pool of external IPs for NAT |
| **Gateway Node** | Node that handles egress traffic |

## Prerequisites

| Component | Version | Notes |
|-----------|---------|-------|
| **Kubernetes** | 1.21+ | Tested on 1.28, 1.29 |
| **Antrea** | 1.14+ | Egress GA since 1.10 |
| **Open vSwitch** | 2.13+ | Included with Antrea |
| **Nodes** | 2+ | At least one gateway node |

### Verify Antrea Installation

```bash
# Check Antrea version
kubectl get pods -n kube-system -l app=antrea -o jsonpath='{.items[0].spec.containers[0].image}'

# Verify Antrea components
kubectl get pods -n kube-system -l app=antrea

# Expected output:
# NAME                    READY   STATUS
# antrea-agent-xxxxx      2/2     Running
# antrea-controller-xxxxx 1/1     Running

# Check Antrea feature gates
kubectl get configmap antrea-config -n kube-system -o yaml | \
  grep -A10 featureGates
```

## Step 1: Enable Egress Feature

### Check Current Configuration

```bash
# View Antrea configuration
kubectl get configmap antrea-config -n kube-system -o yaml
```

### Enable Egress FeatureGate

Edit the Antrea ConfigMap or use Helm:

```bash
# Using Helm (recommended for new installations)
helm upgrade antrea antrea/antrea \
  -n kube-system \
  --set featureGates.Egress=true \
  --set featureGates.ExternalNode=true \
  --wait
```

### Manual Configuration

```yaml
# Edit Antrea ConfigMap
kubectl edit configmap antrea-config -n kube-system

# Add/modify featureGates:
apiVersion: v1
kind: ConfigMap
metadata:
  name: antrea-config
  namespace: kube-system
data:
  antrea.yaml: |
    featureGates:
      Egress: true
      ExternalNode: true
```

### Verify Feature is Enabled

```bash
# Restart Antrea pods to apply changes
kubectl rollout restart daemonset antrea-agent -n kube-system
kubectl rollout restart deployment antrea-controller -n kube-system

# Wait for pods to be ready
kubectl wait --for=condition=Ready pods -l app=antrea -n kube-system --timeout=2m

# Verify Egress is enabled
kubectl exec -n kube-system antrea-agent-xxxxx -- antctl get featuregates | grep Egress
```

## Step 2: Configure Egress IP Pool

Create an `ExternalIPPool` for egress IPs:

```yaml
apiVersion: crd.antrea.io/v1beta1
kind: ExternalIPPool
metadata:
  name: egress-pool-1
spec:
  # Range of IPs for egress NAT
  ipRanges:
    - start: 192.168.100.10
      end: 192.168.100.20
  
  # Nodes that can serve as egress gateways
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/egress-gateway: ""
  
  # Optional: Auto-assign IPs from pool
  autoAssign: true
```

### Apply IP Pool

```bash
kubectl apply -f external-ippool.yaml
```

### Verify IP Pool

```bash
# List external IP pools
kubectl get externalippool

# Get detailed status
kubectl describe externalippool egress-pool-1

# Check allocated IPs
kubectl get externalippool egress-pool-1 -o yaml
```

## Step 3: Create Egress Resource

Define which traffic should use the egress gateway:

```yaml
apiVersion: crd.antrea.io/v1beta1
kind: Egress
metadata:
  name: egress-production
spec:
  # Reference the IP pool
  externalIPPool: egress-pool-1
  
  # Select pods/namespaces that use this egress
  appliedTo:
    - namespaceSelector:
        matchLabels:
          name: production
      podSelector:
        matchLabels:
          app: api-server
  
  # Optional: Specific external destinations
  # If not specified, all external traffic uses egress
  destination:
    - ipBlock:
        cidr: 0.0.0.0/0
        # Exclude internal networks
        except:
          - 10.0.0.0/8
          - 172.16.0.0/12
          - 192.168.0.0/16
```

### Apply Egress Resource

```bash
kubectl apply -f egress.yaml
```

### Verify Egress Status

```bash
# List egress resources
kubectl get egress

# Get detailed status
kubectl describe egress egress-production

# Check which gateway node is assigned
kubectl get egress egress-production -o jsonpath='{.status.egressIP}'
kubectl get egress egress-production -o jsonpath='{.status.gatewayNode}'
```

## Step 4: Label Gateway Nodes

Designate which node(s) will serve as egress gateway:

```bash
# Label a node as egress gateway
kubectl label node worker-node-2 \
  node-role.kubernetes.io/egress-gateway=

# Verify label
kubectl get nodes -l node-role.kubernetes.io/egress-gateway=
```

### High Availability Configuration

For HA, label multiple nodes:

```bash
# Label multiple gateway nodes
kubectl label node worker-node-2 \
  node-role.kubernetes.io/egress-gateway=
kubectl label node worker-node-3 \
  node-role.kubernetes.io/egress-gateway=

# Antrea will automatically failover if primary gateway fails
```

## Step 5: Test Egress Gateway

### Create Test Pod

```bash
# Create test namespace
kubectl create namespace test-egress
kubectl label namespace test-egress name=test-egress

# Deploy test pod
kubectl run test-pod --image=curlimages/curl -it --rm --restart=Never \
  --namespace test-egress \
  --labels app=test-app \
  -- curl -s ifconfig.me
```

### Create Egress for Test

```yaml
apiVersion: crd.antrea.io/v1beta1
kind: Egress
metadata:
  name: egress-test
spec:
  externalIPPool: egress-pool-1
  appliedTo:
    - namespaceSelector:
        matchLabels:
          name: test-egress
      podSelector:
        matchLabels:
          app: test-app
```

### Verify Source IP

```bash
# From pod with egress
kubectl exec -it test-pod -n test-egress -- curl -s ifconfig.me
# Should show egress IP from pool (e.g., 192.168.100.10)

# From pod without egress
kubectl run normal-pod --image=curlimages/curl -it --rm --restart=Never \
  -- curl -s ifconfig.me
# Should show node IP (not from egress pool)
```

## Advanced Configuration

### Multiple Egress Pools

Create different pools for different purposes:

```yaml
# Production egress pool
apiVersion: crd.antrea.io/v1beta1
kind: ExternalIPPool
metadata:
  name: egress-pool-production
spec:
  ipRanges:
    - start: 192.168.100.10
      end: 192.168.100.20
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/egress-gateway: ""

---
# Staging egress pool
apiVersion: crd.antrea.io/v1beta1
kind: ExternalIPPool
metadata:
  name: egress-pool-staging
spec:
  ipRanges:
    - start: 192.168.100.30
      end: 192.168.100.40
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/egress-gateway: ""
```

### Namespace-Based Egress

Apply egress to entire namespace:

```yaml
apiVersion: crd.antrea.io/v1beta1
kind: Egress
metadata:
  name: egress-namespace-wide
spec:
  externalIPPool: egress-pool-production
  appliedTo:
    - namespaceSelector:
        matchLabels:
          name: production
  # No podSelector = all pods in namespace
```

### ExternalNode Support

Antrea supports ExternalNode for VMs/bare metal:

```yaml
apiVersion: crd.antrea.io/v1alpha1
kind: ExternalNode
metadata:
  name: vm-server-1
  namespace: external
spec:
  interfaces:
    - interfaceName: eth0
      ips:
        - 192.168.50.10/24
  nodeAnnotations:
    type: "vm"
```

### Gateway Failover Configuration

```yaml
apiVersion: crd.antrea.io/v1beta1
kind: Egress
metadata:
  name: egress-ha
spec:
  externalIPPool: egress-pool-1
  appliedTo:
    - namespaceSelector:
        matchLabels:
          name: production
  
  # Antrea automatically handles failover
  # When primary gateway fails, secondary takes over
  # No additional configuration needed
```

## Network Policy Integration

### Restrict Egress Traffic

Combine Egress with NetworkPolicy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
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
    
    # Allow only specific external services
    - to:
        - ipBlock:
            cidr: 52.0.0.0/8  # AWS
      ports:
        - protocol: TCP
          port: 443
    
    # All other traffic goes through egress gateway
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
```

### Antrea Native Policy

Use Antrea's enhanced NetworkPolicy:

```yaml
apiVersion: crd.antrea.io/v1beta1
kind: ClusterNetworkPolicy
metadata:
  name: egress-control
spec:
  priority: 10
  appliedTo:
    - podSelector:
        matchLabels:
          app: api-server
  egress:
    - action: Allow
      to:
        - ipBlock:
            cidr: 52.0.0.0/8  # AWS only
      protocols:
        - protocol: TCP
          port: 443
    - action: Drop
      to:
        - ipBlock:
            cidr: 0.0.0.0/0
```

## Monitoring and Observability

### Antrea Controller Metrics

```bash
# Enable metrics in Antrea config
kubectl edit configmap antrea-config -n kube-system

# Add:
# flowExporter:
#   enable: true
#   flowCollectorAddr: "flow-collector:4739"

# Access Prometheus metrics
kubectl port-forward -n kube-system svc/antrea-metrics 9090:9090
```

### Key Metrics

| Metric | Description |
|--------|-------------|
| `antrea_egress_ip_allocations` | Number of allocated egress IPs |
| `antrea_egress_gateway_packets` | Packets processed by egress gateway |
| `antrea_egress_gateway_bytes` | Bytes processed by egress gateway |
| `antrea_egress_gateway_flows` | Active flow count |

### Grafana Dashboard

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: antrea-egress-dashboard
  namespace: monitoring
data:
  antrea-egress.json: |
    {
      "dashboard": {
        "title": "Antrea Egress Gateway",
        "panels": [
          {
            "title": "Egress IP Usage",
            "targets": [
              {
                "expr": "antrea_egress_ip_allocations"
              }
            ]
          },
          {
            "title": "Egress Traffic",
            "targets": [
              {
                "expr": "rate(antrea_egress_gateway_bytes[5m])"
              }
            ]
          }
        ]
      }
    }
```

### Flow Exporter

```yaml
# Enable flow exporter for detailed traffic analysis
helm upgrade antrea antrea/antrea \
  -n kube-system \
  --set featureGates.FlowExporter=true \
  --set flowExporter.enable=true \
  --set flowExporter.flowCollectorAddr="flow-collector.monitoring:4739" \
  --wait
```

## Troubleshooting

### Issue: Egress IP Not Assigned

```bash
# Check ExternalIPPool status
kubectl get externalippool

# Verify node selector matches
kubectl get nodes --show-labels | grep egress-gateway

# Check Egress resource status
kubectl describe egress egress-production

# Look for events
kubectl get events -n kube-system | grep -i egress
```

### Issue: Traffic Not Using Egress

```bash
# Verify pod labels match Egress spec
kubectl get pods -n production --show-labels

# Check if Egress is applied
kubectl get egress -A

# Test from inside pod
kubectl exec -it test-pod -n production -- \
  curl -s ifconfig.me

# Check OVS flows on gateway node
kubectl debug node/gateway-node -it --image=antrea/antrea-agent-ubuntu -- \
  ovs-ofctl dump-flows br-int
```

### Issue: Gateway Node Failure

```bash
# Check gateway node status
kubectl get nodes

# View egress status after failover
kubectl get egress egress-production -o yaml

# Check which node is now handling egress
kubectl get egress egress-production -o jsonpath='{.status.gatewayNode}'

# View Antrea agent logs
kubectl logs -n kube-system -l app=antrea-agent | grep -i egress
```

### Common Problems and Solutions

| Problem | Cause | Solution |
|---------|-------|----------|
| Egress IP not allocated | No gateway nodes labeled | `kubectl label node <name> node-role.kubernetes.io/egress-gateway=` |
| Pod not using egress | Labels don't match Egress spec | Verify pod/namespace labels |
| ExternalIPPool empty | IP range exhausted | Expand IP range or add more pools |
| Failover not working | Only one gateway node | Label additional gateway nodes |
| OVS flow errors | Antrea agent issue | Restart antrea-agent pod |

## Performance Considerations

### Gateway Node Sizing

| Workload | CPU | Memory | Network |
|----------|-----|--------|---------|
| Small (< 50 pods) | 2 cores | 4GB | 1Gbps |
| Medium (50-200 pods) | 4 cores | 8GB | 10Gbps |
| Large (200+ pods) | 8 cores | 16GB | 10Gbps+ |

### Optimization Tips

1. **Use dedicated gateway nodes** - Don't run workloads on egress nodes
2. **Enable IPsec only when needed** - Adds encryption overhead
3. **Tune OVS parameters** - Adjust flow table sizes
4. **Monitor flow expiration** - Prevent flow table exhaustion

```bash
# Check OVS flow statistics
kubectl debug node/gateway-node -it --image=antrea/antrea-agent-ubuntu -- \
  ovs-ofctl dump-flows br-int table=0

# Check Antrea agent metrics
kubectl port-forward -n kube-system svc/antrea-agent 9091:9091
curl localhost:9091/metrics | grep egress
```

## Security Best Practices

### 1. Restrict Egress by Destination

```yaml
apiVersion: crd.antrea.io/v1beta1
kind: Egress
metadata:
  name: restricted-egress
spec:
  externalIPPool: egress-pool-1
  appliedTo:
    - namespaceSelector:
        matchLabels:
          name: production
  destination:
    - ipBlock:
        cidr: 52.0.0.0/8      # AWS only
    - ipBlock:
        cidr: 35.0.0.0/8      # GCP only
```

### 2. Use Separate Pools per Environment

```yaml
# Production - dedicated IPs
apiVersion: crd.antrea.io/v1beta1
kind: ExternalIPPool
metadata:
  name: egress-production
spec:
  ipRanges:
    - start: 192.168.100.10
      end: 192.168.100.20

---
# Staging - separate IPs
apiVersion: crd.antrea.io/v1beta1
kind: ExternalIPPool
metadata:
  name: egress-staging
spec:
  ipRanges:
    - start: 192.168.100.30
      end: 192.168.100.40
```

### 3. Audit Egress Traffic

```yaml
# Enable flow logging for audit
apiVersion: crd.antrea.io/v1alpha1
kind: Traceflow
metadata:
  name: egress-trace
spec:
  source:
    namespace: production
    pod: api-server-xxxxx
  destination:
    ip: "8.8.8.8"
  packet:
    ipHeader:
      protocol: 6  # TCP
      ttl: 128
```

## Comparison with Other Solutions

| Feature | Antrea | Cilium | Istio |
|---------|--------|--------|-------|
| **Egress Gateway** | ✅ Open source | ✅ Open source | ✅ Open source |
| **CNI Type** | OVS-based | eBPF-based | Sidecar proxy |
| **Performance** | Good | Excellent | Good |
| **ExternalNode** | ✅ Supported | ❌ Limited | ❌ No |
| **Network Policy** | ✅ Enhanced NP | ✅ Cilium NP | ✅ Authorization Policy |
| **Encryption** | ✅ IPsec/WireGuard | ❌ No | ✅ mTLS |
| **Complexity** | Medium | Medium | High |
| **Resource Usage** | Low | Low | High (sidecars) |

### When to Choose Antrea

**Choose Antrea when:**
- ✅ You want open source egress gateway (unlike Calico)
- ✅ OVS-based networking is acceptable
- ✅ You need ExternalNode support (VMs/bare metal)
- ✅ IPsec encryption is required
- ✅ Simpler than service mesh (vs Istio)

**Consider alternatives when:**
- 📋 You need eBPF performance (choose Cilium)
- 📋 You want full service mesh (choose Istio)
- 📋 You're already using Calico Enterprise

## Migration from Other CNIs

### From Calico to Antrea

```bash
# 1. Install Antrea alongside Calico (dual-stack temporarily)
helm install antrea antrea/antrea \
  -n kube-system \
  --create-namespace

# 2. Configure egress in Antrea
kubectl apply -f external-ippool.yaml
kubectl apply -f egress.yaml

# 3. Test egress functionality
kubectl run test --image=curlimages/curl -- curl ifconfig.me

# 4. Remove Calico (after validation)
kubectl delete -f calico.yaml
```

### From Flannel to Antrea

```bash
# Note: Requires cluster recreation or careful migration
# 1. Backup all resources
kubectl get all -A -o yaml > backup.yaml

# 2. Install Antrea
helm install antrea antrea/antrea -n kube-system

# 3. Remove Flannel
kubectl delete -f flannel.yaml

# 4. Restore workloads
kubectl apply -f backup.yaml
```

## Next Steps

In the next post of this series:

- **Kube-OVN Egress Gateway** - OVN-based CNI with Floating IP support
- Comparison with Antrea and Cilium
- When to choose each solution

## Conclusion

Antrea Egress Gateway provides:

**Advantages:**
- ✅ Fully open source (unlike Calico Egress)
- ✅ OVS-based packet processing
- ✅ ExternalNode support for VMs/bare metal
- ✅ IPsec and WireGuard encryption
- ✅ Enhanced NetworkPolicy CRDs
- ✅ Lower resource usage than service mesh

**Considerations:**
- 📋 Requires replacing existing CNI
- 📋 OVS adds kernel module dependency
- 📋 Performance slightly below eBPF (Cilium)
- 📋 Smaller community than Cilium/Calico

For organizations needing a **truly open source egress gateway** without vendor lock-in, Antrea is an excellent choice that doesn't require enterprise licensing.

