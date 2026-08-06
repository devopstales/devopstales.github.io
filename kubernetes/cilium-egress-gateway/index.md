---
title: Kubernetes Egress Gateway Options with Cilium
url: https://devopstales.github.io/kubernetes/cilium-egress-gateway/
date: 2026-04-02
keywords: Kubernetes egress, Cilium egress gateway, eBPF networking, Kubernetes security, network policy, egress control, NAT gateway
---


Controlling outbound traffic from your Kubernetes cluster is critical for security, compliance, and audit requirements. This post covers **Cilium Egress Gateway** - a powerful open source solution built on eBPF.

<!--more-->

{{< content "/filedir/egress-gateway-series.html" >}}

## Why Egress Gateway?

Before diving into implementation, let's understand why you need egress control:

```bash
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                        │
│  │  Pod A  │ │  Pod B  │ │  Pod C  │  ← Workloads           │
│  └────┬────┘ └────┬────┘ └────┬────┘                        │
│       │           │           │                             │
│       │           │           │                             │
│       └───────────┼───────────┘                             │
│                   │                                         │
│            ┌──────▼──────┐                                  │
│            │   Egress    │  ← Single exit point             │
│            │   Gateway   │     - Fixed source IP            │
│            └──────┬──────┘     - Audit logging              │
│                   │           - Policy enforcement          │
└───────────────────┼─────────────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────┐
         │  External API    │
         │  (Stripe, AWS)   │  ← Only accepts traffic from
         └──────────────────┘     whitelisted IPs
```

### Common Use Cases

| Requirement | Solution |
|-------------|----------|
| **Fixed source IP** | Egress gateway provides static IP for whitelisting |
| **Compliance** | Audit all outbound traffic through single point |
| **Security policies** | Enforce egress rules per namespace/team |
| **Cost control** | Monitor and limit external API calls |
| **Data residency** | Route traffic through specific regions |

## Cilium Egress Gateway Architecture

Cilium's egress gateway uses eBPF to intercept and redirect outbound traffic:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                           │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Pod with egressGateway: true                           │   │
│   │                                                         │   │
│   │              Outbound traffic                           │   │
│   └─────────────────────┬───────────────────────────────────┘   │
│                         │                                       │
│                         ▼                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Cilium Agent eBPF                                      │   │
│   │                                                         │   │
│   │         Redirect via eBPF                               │   │
│   └─────────────────────┬───────────────────────────────────┘   │
│                         │                                       │
│                         ▼                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Egress Gateway Node                                    │   │
│   │                                                         │   │
│   │         NAT/Masquerade                                  │   │
│   └─────────────────────┬───────────────────────────────────┘   │
│                         │                                       │
│                         │ Fixed IP                              │
└─────────────────────────┼───────────────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  External Service     │
              └───────────────────────┘
```

### How It Works

1. **Pod Label Selection**: Pods with `egressGateway: true` label are selected
2. **eBPF Interception**: Cilium's eBPF program intercepts outbound packets
3. **Traffic Redirect**: Packets are redirected to designated gateway node
4. **Source NAT**: Gateway node masquerades traffic with its IP
5. **Policy Enforcement**: CiliumNetworkPolicy controls which traffic uses gateway

## Prerequisites

| Component | Version | Notes |
|-----------|---------|-------|
| **Kubernetes** | 1.25+ | Tested on 1.28, 1.29 |
| **Cilium** | 1.14+ | Egress Gateway GA since 1.13 |
| **Kernel** | 5.4+ | eBPF support required |
| **Nodes** | 2+ | Dedicated gateway node recommended |

### Verify Cilium Installation

```bash
# Check Cilium version
cilium version

# Verify Cilium is running
kubectl get pods -n kube-system -l k8s-app=cilium

# Check Cilium status
cilium status --wait

# Verify egress gateway is available
cilium config view | grep egress
```

## Step 1: Enable Egress Gateway Feature

### Check Current Configuration

```bash
# View Cilium configuration
kubectl -n kube-system get configmap cilium-config -o yaml | \
  grep -A5 -B5 egress
```

### Enable Egress Gateway (if not enabled)

For existing installations, enable via Helm:

```bash
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --set egressGateway.enabled=true \
  --set ipam.mode=cluster-pool \
  --set cluster.name=default \
  --wait
```

**Key Configuration Options:**

| Parameter | Value | Description |
|-----------|-------|-------------|
| `egressGateway.enabled` | `true` | Enable egress gateway feature |
| `ipam.mode` | `cluster-pool` | Required for egress gateway |
| `cluster.name` | `default` | Cluster identifier |

### Verify Feature is Enabled

```bash
# Check Cilium daemonset has correct args
kubectl -n kube-system get daemonset cilium -o yaml | \
  grep -A2 egress-gateway

# Expected output:
# - --enable-egress-gateway
# - --cluster-name=default
```

## Step 2: Label Gateway Node

Designate which node(s) will serve as egress gateway:

```bash
# List available nodes
kubectl get nodes

# Label a node as egress gateway
kubectl label node worker-node-1 \
  cilium.io/egress-gateway=true

# Verify label
kubectl get nodes -l cilium.io/egress-gateway=true
```

### Best Practices for Node Selection

| Criteria | Recommendation |
|----------|----------------|
| **Dedicated node** | Use separate node for gateway (not running workloads) |
| **High availability** | Label multiple nodes for redundancy |
| **Network placement** | Choose node with direct external access |
| **Resources** | Ensure sufficient CPU/memory for NAT operations |

## Step 3: Configure Static Egress IP

The **egressIP** is the predictable source IP that external services will see. This is critical for allowlisting with external APIs, payment processors, and partner services.

### Option 1: Specify Static Egress IP

```yaml
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: egress-gateway-policy
spec:
  selectors:
  - podSelector:
      matchLabels:
        use-egress-gateway: "true"
      namespaceSelector:
        matchLabels:
          name: production
  
  destinationCIDRs:
  - "0.0.0.0/0"
  
  egressGateway:
    nodeSelector:
      matchLabels:
        cilium.io/egress-gateway: "true"
    egressIP: 192.168.1.100  # Static IP for outbound traffic
```

**Important:** The egressIP must be pre-configured on the gateway node interface.

### Option 2: Use Interface IP (First IP on Interface)

```yaml
egressGateway:
  nodeSelector:
    matchLabels:
      cilium.io/egress-gateway: "true"
  interface: eth1  # First IPv4/IPv6 on this interface will be used
```

### Option 3: Default Route Interface (Automatic)

```yaml
egressGateway:
  nodeSelector:
    matchLabels:
      cilium.io/egress-gateway: "true"
# Uses first IPv4/IPv6 on the default route interface
```

### Configure Static IP on Gateway Node

```bash
# Add secondary IP to gateway node interface
sudo ip addr add 192.168.1.100/32 dev eth1

# Verify IP is configured
ip addr show eth1

# Make persistent (Ubuntu/Debian)
cat >> /etc/network/interfaces << EOF
auto eth1:1
iface eth1:1 inet static
    address 192.168.1.100
    netmask 255.255.255.255
EOF

# Make persistent (RHEL/CentOS)
cat >> /etc/sysconfig/network-scripts/ifcfg-eth1:1 << EOF
DEVICE=eth1:1
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.1.100
NETMASK=255.255.255.255
EOF
```

### Cloud Provider Static IP Configuration

#### AWS EKS - Elastic IP

```bash
# Allocate Elastic IP
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)

# Get gateway node ENI ID
ENI_ID=$(aws ec2 describe-instances \
  --instance-ids $(kubectl get node gateway-node -o jsonpath='{.spec.providerID}' | cut -d'/' -f5) \
  --query 'Reservations[0].Instances[0].NetworkInterfaces[0].NetworkInterfaceId' \
  --output text)

# Associate EIP with ENI
aws ec2 associate-address \
  --allocation-id $EIP_ALLOC \
  --network-interface-id $ENI_ID

# Get the Elastic IP
EIP=$(aws ec2 describe-addresses \
  --allocation-ids $EIP_ALLOC \
  --query 'Addresses[0].PublicIp' \
  --output text)

echo "Configure egressIP: $EIP in your policy"
```

#### GCP - Alias IP Range

```bash
# Reserve static external IP
gcloud compute addresses create egress-ip --region us-central1

# Get IP address
EGRESS_IP=$(gcloud compute addresses describe egress-ip \
  --region us-central1 \
  --format="value(address)")

# Add alias IP to gateway instance
gcloud compute instances update-network-interface \
  gateway-node \
  --network-interface primary \
  --aliases $EGRESS_IP/32 \
  --zone us-central1-a
```

#### Azure - Secondary IP Configuration

```bash
# Get NIC ID
NIC_ID=$(az vm show \
  -g my-aks-rg \
  -n gateway-node \
  --query 'networkProfile.networkInterfaces[0].id' \
  --output tsv)

# Create public IP
az network public-ip create \
  -g my-aks-rg \
  -n egress-ip \
  --sku Standard \
  --allocation-method Static

# Get public IP
EGRESS_IP=$(az network public-ip show \
  -g my-aks-rg \
  -n egress-ip \
  --query 'ipAddress' \
  --output tsv)

# Associate with NIC (requires additional NIC configuration)
# See Azure documentation for secondary IP configuration
```

### Multiple Egress IPs for Different Services

```yaml
# Payment services - dedicated egress IP
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: egress-payments
spec:
  selectors:
  - podSelector:
      matchLabels:
        app: payment-service
  destinationCIDRs:
  - "0.0.0.0/0"
  egressGateway:
    nodeSelector:
      matchLabels:
        cilium.io/egress-gateway: "true"
    egressIP: 192.168.1.100  # Payment IP (allowlisted with payment processor)

---
# Analytics services - separate egress IP
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: egress-analytics
spec:
  selectors:
  - podSelector:
      matchLabels:
        app: analytics-service
  destinationCIDRs:
  - "0.0.0.0/0"
  egressGateway:
    nodeSelector:
      matchLabels:
        cilium.io/egress-gateway: "true"
    egressIP: 10.168.60.101  # Analytics IP (allowlisted with analytics provider)
```

## Step 4: Create Egress Gateway Policy

Create `egress-gateway-policy.yaml`:

```yaml
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: egress-gateway-policy
  namespace: production
spec:
  # Select pods that should use egress gateway
  selector:
    matchLabels:
      use-egress-gateway: "true"
  
  # Destination CIDRs (external services)
  destinationCIDRs:
    - "0.0.0.0/0"  # All external traffic
    # Or specific ranges:
    # - "52.0.0.0/8"    # AWS
    # - "35.0.0.0/8"    # GCP
  
  # Excluded CIDRs (internal traffic - don't NAT)
  excludedCIDRs:
    - "10.0.0.0/8"     # Cluster internal
    - "172.16.0.0/12"  # Private networks
    - "192.168.0.0/16" # Private networks
  
  # Gateway node selection
  egressGateway:
    nodeSelector:
      matchLabels:
        cilium.io/egress-gateway: "true"
    egressIP: "192.168.1.100"
```

### Apply the Policy

```bash
kubectl apply -f egress-gateway-policy.yaml
```

### Verify Policy

```bash
# List egress gateway policies
kubectl get CiliumEgressGatewayPolicy -A

# Get policy details
kubectl describe CiliumEgressGatewayPolicy egress-gateway-policy -n production

# Check enforcement status
cilium policy get --all | grep -A10 EgressGateway
```

## Step 4: Label Pods for Egress Gateway

Label pods that should use the egress gateway:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: egress-pod
  namespace: production
  labels:
    use-egress-gateway: "true"  # This triggers egress gateway
spec:
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80
```

### Apply to Deployments

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
      use-egress-gateway: "true"  # ← Important label
  template:
    metadata:
      labels:
        app: api-server
        use-egress-gateway: "true"  # ← Important label
    spec:
      containers:
      - name: api
        image: myapp/api:v1.0
        ports:
        - containerPort: 8080
```

### Apply Labels to Existing Deployments

```bash
# Label existing deployment
kubectl label deployment api-server -n production \
  use-egress-gateway=true

# This won't work - you need to label the pod template
kubectl patch deployment api-server -n production --type='json' \
  -p='[{"op": "add", "path": "/spec/template/metadata/labels/use-egress-gateway", "value": "true"}]'

# Restart pods to apply new labels
kubectl rollout restart deployment api-server -n production
```

## Step 5: Test Egress Gateway

### Create Test Pod

```bash
# Deploy test pod
kubectl run test-egress --image=curlimages/curl -it --rm --restart=Never \
  --namespace production \
  --labels use-egress-gateway=true \
  -- curl -s ifconfig.me
```

### Verify Source IP

```bash
# From inside the pod (with egress gateway)
kubectl exec -it test-egress -n production -- curl -s ifconfig.me
# Should show gateway node's external IP

# From a pod without egress gateway
kubectl run test-normal --image=curlimages/curl -it --rm --restart=Never \
  -- curl -s ifconfig.me
# Should show node's IP (not gateway)
```

### Compare Traffic Paths

```bash
# Enable monitoring on gateway node
kubectl -n kube-system exec -it cilium-xxxxx -- cilium monitor --type drop

# Watch NAT translations
kubectl -n kube-system exec -it cilium-xxxxx -- cilium bpf nat list
```

## Advanced Configuration

### High Availability with L2 Announcements

Cilium L2 Announcements provide automatic failover for egress IPs without requiring external tools like keepalived.

#### Enable L2 Announcements

```bash
# Enable L2 announcements via Helm
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set l2announcements.enabled=true \
  --set l2announcements.leaseDuration=3s \
  --set l2announcements.leaseRenewDeadline=1s \
  --set l2announcements.leaseRetryPeriod=200ms \
  --set k8sClientRateLimit.qps=50 \
  --set k8sClientRateLimit.burst=100

# Verify L2 announcements are enabled
kubectl -n kube-system exec ds/cilium -- cilium-dbg config --all | grep EnableL2Announcements
```

#### Configure L2 Announcement Policy

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: egress-l2-policy
spec:
  # Select which nodes can announce egress IPs
  nodeSelector:
    matchLabels:
      cilium.io/egress-gateway: "true"
  
  # Specify interfaces for L2 announcements
  interfaces:
  - ^eth[0-9]+
  
  # Announce loadBalancer IPs (for LoadBalancer services)
  loadBalancerIPs: true
  
  # Announce external IPs
  externalIPs: true
```

#### Failover Tuning

| Parameter | Default | Fast Failover | Description |
|-----------|---------|---------------|-------------|
| `leaseDuration` | 15s | 3s | Time before failover occurs |
| `leaseRenewDeadline` | 5s | 1s | Leader renew interval |
| `leaseRetryPeriod` | 2s | 200ms | Retry period on failure |

**Failover Time:** With fast failover settings, expect 2-4 seconds failover time.

### LoadBalancer Service for Egress IP

For high availability with automatic IP failover, use a LoadBalancer service:

#### Option 1: Cilium L2 Aware Load Balancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: egress-lb
  namespace: kube-system
  labels:
    cilium.io/l2-lb: "true"
  annotations:
    # Specify loadBalancerClass for L2 announcer
    loadBalancerClass: io.cilium/l2-announcer
spec:
  type: LoadBalancer
  # Assign static IP from your pool
  loadBalancerIP: 192.168.1.100
  # Important: Use Cluster mode (Local is incompatible with L2)
  externalTrafficPolicy: Cluster
  selector:
    cilium.io/egress-gateway: "true"
  ports:
  - name: egress-https
    port: 443
    targetPort: 443
    protocol: TCP
```

#### Option 2: MetalLB L2 Service

If using MetalLB instead of Cilium L2:

```yaml
# MetalLB IPAddressPool
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: egress-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.100-192.168.1.110  # Pool of egress IPs
  autoAssign: false

---
# MetalLB L2Advertisement
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: egress-l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - egress-pool
  nodeSelectors:
  - matchLabels:
      cilium.io/egress-gateway: "true"

---
# LoadBalancer Service for egress
apiVersion: v1
kind: Service
metadata:
  name: egress-lb
  namespace: kube-system
  annotations:
    metallb.universe.tf/loadBalancerIPs: 192.168.1.100
spec:
  type: LoadBalancer
  externalTrafficPolicy: Cluster
  selector:
    cilium.io/egress-gateway: "true"
  ports:
  - port: 443
    targetPort: 443
```

#### Verify LoadBalancer Status

```bash
# Check LoadBalancer service
kubectl get svc egress-lb -n kube-system

# Expected output:
# NAME       TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)
# egress-lb  LoadBalancer   10.96.x.x     192.168.1.100  443:31234/TCP

# Verify L2 announcements
kubectl get CiliumL2AnnouncementPolicy

# Check which node is the leader
kubectl -n kube-system get lease | grep cilium-l2announce
```

### Multiple Gateway Nodes with L2

```yaml
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: ha-egress-policy
spec:
  selectors:
  - podSelector:
      matchLabels:
        use-egress-gateway: "true"
  
  destinationCIDRs:
  - "0.0.0.0/0"
  
  # Multiple gateway nodes for HA
  egressGateways:
  - nodeSelector:
      matchLabels:
        cilium.io/egress-gateway: "true"
        topology.kubernetes.io/zone: us-east-1a
    egressIP: 192.168.1.100
  
  - nodeSelector:
      matchLabels:
        cilium.io/egress-gateway: "true"
        topology.kubernetes.io/zone: us-east-1b
    egressIP: 192.168.1.101
```

**Note:** Each endpoint uses a single gateway based on CiliumEndpoint UID. Cilium distributes traffic across available gateways.

### Per-Namespace Egress Control

```yaml
# Namespace with strict egress
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    egress-policy: strict

---
# Policy for secure namespace
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: strict-egress
  namespace: secure-namespace
spec:
  selector:
    matchLabels: {}  # All pods in namespace
  
  destinationCIDRs:
    - "10.0.0.0/8"      # Allow internal
    - "52.0.0.0/8"      # Allow AWS only
  
  egressGateway:
    nodeSelector:
      matchLabels:
        cilium.io/egress-gateway: "true"
```

### Combine with CiliumNetworkPolicy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: egress-restrict
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      use-egress-gateway: "true"
  
  egress:
    # Allow DNS
    - toEndpoints:
        - matchLabels:
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
          protocol: UDP
    
    # Allow only specific external services via gateway
    - toCIDR:
        - "52.0.0.0/8"     # AWS
        - "35.0.0.0/8"     # GCP
      toPorts:
        - ports:
            - port: "443"
          protocol: TCP
```

## Monitoring and Observability

### Enable Hubble for Egress Monitoring

```bash
# Enable Hubble (if not already enabled)
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --wait
```

### View Egress Traffic

```bash
# Port-forward Hubble UI
kubectl port-forward -n kube-system svc/hubble-ui 8080:80

# Or use Hubble CLI
hubble observe --namespace production --follow

# Filter for egress gateway traffic
hubble observe --verdict FORWARDED --policy-verdict EGRESS_GATEWAY
```

### Metrics and Dashboards

```yaml
# Prometheus metrics available:
# - cilium_egress_gateway_enabled
# - cilium_egress_gateway_nat_entries
# - cilium_egress_gateway_packets_total
# - cilium_egress_gateway_bytes_total
```

```
                    ┌─────────────────────────┐
                    │ Cilium Egress Gateway   │
                    └───────────┬─────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
     ┌─────────────────┐ ┌─────────────┐ ┌─────────────────┐
     │   Prometheus    │ │   Hubble    │ │ Cilium Status   │
     └────────┬────────┘ └──────┬──────┘ └────────┬────────┘
              │                 │                  │
              ▼                 ▼                  ▼
     ┌─────────────────┐ ┌─────────────┐ ┌─────────────────┐
     │Grafana Dashboard│ │  Hubble UI  │ │kubectl/cilium CLI│
     └─────────────────┘ └─────────────┘ └─────────────────┘
```

## Troubleshooting

### Issue: Egress Gateway Not Working

```bash
# 1. Check if feature is enabled
cilium config view | grep egress

# 2. Verify node label
kubectl get nodes -l cilium.io/egress-gateway=true

# 3. Check policy status
kubectl get CiliumEgressGatewayPolicy -A

# 4. Verify pod labels
kubectl get pods -n production --show-labels | grep egress

# 5. Check Cilium logs
kubectl -n kube-system logs -l k8s-app=cilium | grep -i egress
```

### Issue: Static Egress IP Not Working

```bash
# Verify egressIP is configured on node
kubectl debug node/gateway-node -it --image=cilium/cilium -- ip addr show eth1

# Check egress configuration
kubectl -n kube-system exec ds/cilium -- cilium-dbg bpf egress list

# Expected output:
# Source IP    Destination CIDR    Egress IP      Gateway IP
# 192.168.2.23 192.168.60.13/32    192.168.1.100  192.168.60.12

# Verify routing
kubectl debug node/gateway-node -it --image=cilium/cilium -- ip route get 8.8.8.8 from 192.168.1.100
```

### Issue: L2 Announcements Not Working

```bash
# Verify L2 announcements are enabled
kubectl -n kube-system exec ds/cilium -- cilium-dbg config --all | grep EnableL2Announcements

# Check L2 policies
kubectl get CiliumL2AnnouncementPolicy

# Inspect leases
kubectl -n kube-system get lease | grep cilium-l2announce

# Check agent logs
kubectl -n kube-system logs ds/cilium | grep "l2"

# Inspect L2 announce state
kubectl -n kube-system exec pod/cilium-<id> -- cilium-dbg shell -- db/show l2-announce

# Check known devices
kubectl -n kube-system exec ds/cilium -- cilium-dbg shell -- db/show devices

# Test ARP from within cluster
kubectl -n kube-system exec pod/cilium-<id> -- arping -i eth0 <service-ip>

# Check BPF map for ARP responses
kubectl -n kube-system exec pod/cilium-<id> -- bpftool map dump pinned /sys/fs/bpf/tc/globals/cilium_l2_responder_v4
```

### Issue: LoadBalancer Service Not Getting External IP

```bash
# Check LoadBalancer service status
kubectl get svc egress-lb -n kube-system

# Verify loadBalancerClass annotation
kubectl get svc egress-lb -n kube-system -o yaml | grep loadBalancerClass

# For MetalLB, check IPAddressPool
kubectl get IPAddressPool -n metallb-system

# Check L2Advertisement
kubectl get L2Advertisement -n metallb-system

# Verify node selectors match
kubectl get nodes --show-labels | grep egress-gateway
```

### Issue: Traffic Not Being NAT'd

```bash
# Check NAT table on gateway node
kubectl debug node/gateway-node -it --image=cilium/cilium -- cilium bpf nat list

# Verify eBPF programs
kubectl debug node/gateway-node -it --image=cilium/cilium -- cilium bpf prog list

# Check for drops
kubectl -n kube-system exec -it cilium-xxxxx -- cilium monitor --type drop
```

### Issue: Policy Not Enforcing

```bash
# Get policy UUID
kubectl get CiliumEgressGatewayPolicy -n production -o jsonpath='{.items[0].metadata.uid}'

# Check policy enforcement
cilium policy get | grep -A20 <policy-uuid>

# Verify endpoint labels
cilium endpoint list -n production
```

### Issue: L2 Failover Not Working

```bash
# Check current leader
kubectl -n kube-system get lease cilium-l2announce-<node-name>

# Check failover timing
kubectl -n kube-system logs ds/cilium | grep "leader election"

# Verify lease renew deadline
helm get values cilium -n kube-system | grep leaseRenewDeadline

# Test failover manually
kubectl cordon <current-leader-node>
kubectl drain <current-leader-node> --ignore-daemonsets --delete-emptydir-data

# Watch for new leader
watch kubectl -n kube-system get lease | grep cilium-l2announce
```

### Common Problems and Solutions

| Problem | Cause | Solution |
|---------|-------|----------|
| Gateway not selected | Missing node label | `kubectl label node <name> cilium.io/egress-gateway=true` |
| Pods not using gateway | Missing pod label | Add `use-egress-gateway: "true"` to pod template |
| EgressIP not configured | IP not on interface | `ip addr add <egressIP>/32 dev eth1` |
| L2 not announcing | L2 disabled | Enable with `--set l2announcements.enabled=true` |
| LoadBalancer pending | No IP pool | Create IPAddressPool or allocate static IP |
| Internal traffic NAT'd | Wrong excludedCIDRs | Add cluster CIDR to excludedCIDRs |
| DNS not working | DNS not excluded | Add DNS server IPs to excludedCIDRs |
| Policy not applied | Cilium version mismatch | Upgrade Cilium to 1.13+ |
| L2 failover slow | Default lease settings | Use fast failover (3s/1s/200ms) |
| ARP not responding | Wrong interface | Add interface to `--devices` flag |

## Performance Considerations

### Gateway Node Sizing

| Workload | CPU | Memory | Network |
|----------|-----|--------|---------|
| Small (< 100 pods) | 2 cores | 4GB | 1Gbps |
| Medium (100-500 pods) | 4 cores | 8GB | 10Gbps |
| Large (500+ pods) | 8 cores | 16GB | 10Gbps+ |

### Optimization Tips

1. **Use dedicated gateway nodes** - Don't run workloads on gateway nodes
2. **Enable XDP acceleration** - For high-throughput scenarios
3. **Tune conntrack table** - Increase for many connections
4. **Monitor NAT table size** - Prevent exhaustion

```bash
# Check conntrack usage
cilium bpf ct list global | wc -l

# Increase if needed (via Cilium config)
helm upgrade cilium cilium/cilium \
  --set bpfMaps.ctMapSize=131072 \
  --wait
```

## Security Best Practices

### 1. Restrict Gateway Access

```yaml
# Only allow specific namespaces to use egress gateway
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: restricted-gateway
  namespace: production
spec:
  selector:
    matchLabels:
      use-egress-gateway: "true"
    matchExpressions:
      - key: namespace
        operator: In
        values: ["production", "staging"]
```

### 2. Audit Egress Traffic

```yaml
# Enable logging for egress traffic
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: audit-egress
spec:
  endpointSelector:
    matchLabels:
      use-egress-gateway: "true"
  
  egress:
    - toCIDR:
        - "0.0.0.0/0"
      log: true  # Enable logging
```

### 3. Limit Destination CIDRs

```yaml
# Instead of 0.0.0.0/0, specify allowed destinations
spec:
  destinationCIDRs:
    - "52.0.0.0/8"     # AWS
    - "35.0.0.0/8"     # GCP
    - "13.107.0.0/16"  # Azure
    - "140.82.0.0/16"  # GitHub
```

## Comparison with Other Solutions

| Feature | Cilium | Calico | Cloud NAT |
|---------|--------|--------|-----------|
| **Performance** | eBPF (fastest) | iptables/IPVS | Cloud-native |
| **Policy granularity** | Per-pod | Per-namespace | Per-subnet |
| **Observability** | Hubble UI | Flow logs | Cloud monitoring |
| **Multi-cluster** | Cluster mesh | Federation | VPC peering |
| **Cost** | Free (open source) | Free (open source) | Pay per GB |
| **Complexity** | Medium | Low | Low |

## Next Steps

In the next post of this series, we'll cover:

- **Antrea Egress Gateway** - Open vSwitch-based CNI
- Comparison with Cilium
- When to choose each solution

## Conclusion

Cilium Egress Gateway provides:

**Advantages:**
- ✅ High performance (eBPF-based)
- ✅ Fine-grained policy control (per-pod)
- ✅ Excellent observability (Hubble)
- ✅ Open source (no vendor lock-in)
- ✅ Multi-cluster support (Cluster Mesh)

**Considerations:**
- 📋 Requires Cilium CNI (not compatible with other CNIs)
- 📋 Kernel 5.4+ required for full eBPF features
- 📋 Learning curve for Cilium policies
- 📋 Dedicated gateway nodes recommended

For organizations needing strict egress control with excellent observability, Cilium Egress Gateway is a strong choice in the open source ecosystem.

