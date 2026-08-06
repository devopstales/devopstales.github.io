---
title: Use Multus CNI in Kubernetes
url: https://devopstales.github.io/kubernetes/multus/
date: 2026-03-22
keywords: Kubernetes, Multus CNI, multi-homed pods, network attachment, CNI plugins, pod networking, kubernetes networking
---


Multus CNI enables attaching multiple network interfaces to Kubernetes pods, essential for service mesh, security isolation, and high-performance networking. This updated guide for 2026 covers Multus 4.0+, Kubernetes 1.28+, and modern CNI plugins.

<!--more-->

## What is Multus CNI?

Multus CNI is a **meta-plugin** (container network interface plugin) for Kubernetes that enables attaching **multiple network interfaces** to pods. In standard Kubernetes, each pod has only one network interface (plus loopback). With Multus, you can create **multi-homed pods** with multiple interfaces for different purposes:

- **Secondary networks** for storage traffic
- **Isolated networks** for security segmentation
- **High-performance networks** (SR-IOV, DPDK)
- **Service mesh** sidecar communication
- **Legacy application** requirements

```bash
┌─────────────────────────────────────────────────────────┐
│                      Pod with Multus                    │
│  ┌─────────────┐                                        │
│  │  Container  │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│    ┌────┴────┐                                          │
│    │  eth0   │──► Default Network (pod-to-pod)         │
│    │  net1   │──► Secondary Network (storage)          │
│    │  net2   │──► Tertiary Network (management)        │
│    └─────────┘                                          │
└─────────────────────────────────────────────────────────┘
```

## Architecture Overview

```bash
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Pod       │◄──►│   Multus    │◄──►│  CNI Plugin │
│  (net-ann)  │    │  (meta-CNI) │    │  (macvlan)  │
└─────────────┘    └─────────────┘    └─────────────┘
                          │
                   ┌──────┴──────┐
                   ▼             ▼
            ┌────────────┐ ┌────────────┐
            │  Flannel   │ │  Calico    │
            │  (default) │ │ (secondary)│
            └────────────┘ └────────────┘
```

## Prerequisites

- **Kubernetes 1.28+** cluster
- **Default CNI** already installed (Flannel, Calico, Cilium, etc.)
- **kubectl** configured with cluster access
- **Node network interfaces** for secondary networks

## Step 1: Install Default Network

Multus requires a default CNI for primary pod networking. This guide uses **Flannel** for simplicity, but you can use any CNI.

### Install Flannel

```bash
# Download Flannel manifest
curl -LO https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Edit to specify the correct interface
nano kube-flannel.yml
```

Update the args section to specify your interface:

```yaml
spec:
  containers:
  - name: kube-flannel
    args:
    - --ip-masq
    - --kube-subnet-mgr
    - --iface=enp0s8  # Change to your interface name
```

### Apply Flannel Configuration

```bash
kubectl apply -f kube-flannel.yml

# Verify Flannel pods are running
kubectl get pods -n kube-flannel

# Verify nodes are ready
kubectl get nodes
```

## Step 2: Install Multus

### Download Multus Manifests

```bash
# Get the latest Multus daemonset
curl -LO https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml

# Or use the thick plugin version (recommended for 2026)
curl -LO https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick-plugin.yml
```

### Apply Multus Configuration

```bash
kubectl apply -f multus-daemonset-thick.yml

# Verify Multus pods are running
kubectl get pods -n kube-system | grep multus

# Expected output:
# multus-controller-xxxxx    1/1     Running   0          2m
```

### Validate Installation

```bash
# Check Multus binary exists on nodes
kubectl debug node/<node-name> -it --image=busybox -- chroot /host ls -la /opt/cni/bin/multus

# Verify CNI configuration
kubectl debug node/<node-name> -it --image=busybox -- chroot /host cat /etc/cni/net.d/00-multus.conf
```

## Step 3: Create NetworkAttachmentDefinition

NetworkAttachmentDefinition is a **Custom Resource** that defines additional network configurations.

### Basic Macvlan Configuration

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/resourceName: macvlan-cni-network
spec:
  config: '{
    "cniVersion": "1.0.0",
    "name": "macvlan-network",
    "type": "macvlan",
    "master": "enp0s9",
    "mode": "bridge",
    "ipam": {
      "type": "host-local",
      "subnet": "172.17.9.0/24",
      "rangeStart": "172.17.9.240",
      "rangeEnd": "172.17.9.250",
      "routes": [
        { "dst": "0.0.0.0/0" }
      ],
      "gateway": "172.17.9.1"
    }
  }'
EOF
```

### Configuration Parameters Explained

| Parameter | Description | Required |
|-----------|-------------|----------|
| `cniVersion` | CNI specification version | Yes |
| `type` | CNI plugin binary to execute | Yes |
| `master` | Host interface name for secondary network | Yes |
| `mode` | Macvlan mode (bridge, private, vepa, passthru) | Yes |
| `ipam.type` | IP address management plugin | Yes |
| `subnet` | IP range for secondary network | Yes |
| `rangeStart/End` | Specific IP allocation range | No |
| `gateway` | Default gateway for secondary network | No |

### Verify NetworkAttachmentDefinition

```bash
# List all network attachments
kubectl get network-attachment-definitions

# Get detailed configuration
kubectl get network-attachment-definitions macvlan-conf -o yaml

# Describe for events
kubectl describe network-attachment-definitions macvlan-conf
```

## Step 4: NetworkAttachmentDefinition Types

### Bridge Plugin

Creates a Linux bridge for pod connectivity:

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: bridge-conf
  namespace: default
spec:
  config: '{
    "cniVersion": "1.0.0",
    "name": "bridge-network",
    "type": "bridge",
    "bridge": "br0",
    "isDefaultGateway": false,
    "ipam": {
      "type": "host-local",
      "subnet": "192.168.12.0/24",
      "rangeStart": "192.168.12.10",
      "rangeEnd": "192.168.12.200"
    }
  }'
```

**Use Case:** Isolated pod communication on same node.

### Host-Device Plugin

Moves a physical host interface into the pod:

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: host-device-conf
  namespace: default
spec:
  config: '{
    "cniVersion": "1.0.0",
    "name": "host-device-network",
    "type": "host-device",
    "device": "enp0s9",
    "ipam": {
      "type": "dhcp"
    }
  }'
```

**Use Case:** Direct hardware access, DPDK applications.

**Note:** Requires DHCP server on the network.

### IPvlan Plugin

Creates IPvlan sub-interfaces (shares MAC with parent):

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: ipvlan-conf
  namespace: default
spec:
  config: '{
    "cniVersion": "1.0.0",
    "name": "ipvlan-network",
    "type": "ipvlan",
    "master": "enp0s9",
    "mode": "l2",
    "ipam": {
      "type": "host-local",
      "subnet": "172.17.9.0/24",
      "rangeStart": "172.17.9.201",
      "rangeEnd": "172.17.9.205",
      "gateway": "172.17.9.1"
    }
  }'
```

**Use Case:** Cloud environments with MAC address restrictions.

| Mode | Description |
|------|-------------|
| `l2` | Layer 2 - traffic switched at L2 |
| `l3` | Layer 3 - traffic routed at L3 |
| `l3s` | Layer 3 with source IP filtering |

### Macvlan Plugin

Creates macvlan sub-interfaces (unique MAC per pod):

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
  namespace: default
spec:
  config: '{
    "cniVersion": "1.0.0",
    "name": "macvlan-network",
    "type": "macvlan",
    "master": "enp0s9",
    "mode": "bridge",
    "ipam": {
      "type": "static"
    }
  }'
```

**Use Case:** Traditional network integration, bare-metal deployments.

**Note:** May not work on all cloud platforms due to MAC address restrictions.

### SR-IOV Plugin (Advanced)

For high-performance networking:

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: sriov-conf
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/resourceName: intel.com/sriov
spec:
  config: '{
    "cniVersion": "1.0.0",
    "type": "sriov",
    "ipam": {
      "type": "host-local",
      "subnet": "10.56.217.0/24"
    }
  }'
```

**Use Case:** High-performance workloads, NFV, low-latency applications.

## Step 5: Create Multi-Homed Pods

### Basic Pod with Secondary Interface

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multus-pod-1
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [
        {
          "name": "macvlan-conf",
          "namespace": "default",
          "interface": "net1"
        }
      ]
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot:latest
    command: ["tail"]
    args: ["-f", "/dev/null"]
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
  terminationGracePeriodSeconds: 0
  nodeSelector:
    kubernetes.io/hostname: worker-node-1
EOF
```

### Pod with Multiple Secondary Interfaces

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-network-pod
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [
        {
          "name": "macvlan-conf",
          "namespace": "default",
          "interface": "storage-net"
        },
        {
          "name": "ipvlan-conf",
          "namespace": "default",
          "interface": "mgmt-net"
        }
      ]
spec:
  containers:
  - name: app
    image: nginx:latest
    ports:
    - containerPort: 80
  terminationGracePeriodSeconds: 30
EOF
```

### Pod with Static IP Assignment

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: static-ip-pod
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [
        {
          "name": "macvlan-conf",
          "interface": "net1",
          "ips": ["172.17.9.245/24"],
          "gateway": ["172.17.9.1"]
        }
      ]
spec:
  containers:
  - name: app
    image: busybox:latest
    command: ["sleep", "infinity"]
EOF
```

## Step 6: Verify Network Configuration

### Check Pod Interfaces

```bash
# View all interfaces in the pod
kubectl exec -it multus-pod-1 -- ip addr

# Expected output:
# 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
#     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
#     inet 127.0.0.1/8 scope host lo
# 3: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP
#     link/ether 06:56:cf:cb:3e:75 brd ff:ff:ff:ff:ff:ff link-netnsid 0
#     inet 10.244.0.5/24 brd 10.244.0.255 scope global eth0
# 4: net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN
#     link/ether 08:00:27:a0:41:35 brd ff:ff:ff:ff:ff:ff
#     inet 172.17.9.240/24 brd 172.17.9.255 scope global net1
```

### Test Connectivity

```bash
# Ping from secondary interface
kubectl exec -it multus-pod-1 -- ping -c 3 -I net1 172.17.9.241

# Ping gateway
kubectl exec -it multus-pod-1 -- ping -c 3 -I net1 172.17.9.1

# Test external connectivity
kubectl exec -it multus-pod-1 -- ping -c 3 -I net1 8.8.8.8

# Traceroute from secondary interface
kubectl exec -it multus-pod-1 -- traceroute -i net1 8.8.8.8
```

### View Network Status Annotation

```bash
# Check network status annotation
kubectl get pod multus-pod-1 -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}' | jq
```

## Advanced Configurations

### Default Route Configuration

By default, the primary interface (eth0) handles default routing. To change this:

```yaml
annotations:
  k8s.v1.cni.cncf.io/networks: |
    [
      {
        "name": "macvlan-conf",
        "default-route": ["172.17.9.1"]
      }
    ]
```

### Network Policy with Multus

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: multus-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: multus-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: allowed
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

### Resource Quotas for Secondary Networks

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: multus-quota
  namespace: default
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    # SR-IOV resources
    intel.com/sriov: "4"
```

## Troubleshooting

### Multus Pods Not Starting

```bash
# Check Multus daemonset status
kubectl get daemonset -n kube-system multus

# View Multus pod logs
kubectl logs -n kube-system -l name=multus

# Check CNI configuration on node
kubectl debug node/<node-name> -it --image=busybox -- \
  chroot /host cat /etc/cni/net.d/00-multus.conf
```

### Secondary Interface Not Appearing

```bash
# Verify NetworkAttachmentDefinition exists
kubectl get network-attachment-definitions

# Check pod annotations
kubectl get pod <pod-name> -o yaml | grep -A 20 annotations

# Check pod events
kubectl describe pod <pod-name>
```

### IP Address Not Assigned

```bash
# Check IPAM configuration
kubectl get network-attachment-definitions <name> -o yaml

# Verify IP range is not exhausted
kubectl exec -it <pod-name> -- ip addr show

# Check for IP conflicts
kubectl exec -it <pod-name> -- arping -I net1 <gateway-ip>
```

### Performance Issues

```bash
# Check interface statistics
kubectl exec -it <pod-name> -- ip -s link

# Test bandwidth
kubectl exec -it <pod-name> -- iperf3 -c <target-ip> -I net1

# Check MTU settings
kubectl exec -it <pod-name> -- ip link show
```

## Best Practices for 2026

### Security

1. **Network Segmentation**: Use secondary networks for sensitive workloads
2. **Network Policies**: Apply Kubernetes NetworkPolicy to secondary interfaces
3. **Encryption**: Use IPsec or TLS for inter-pod communication
4. **RBAC**: Restrict NetworkAttachmentDefinition creation to admins

### Performance

1. **SR-IOV**: For latency-sensitive workloads
2. **IPvlan L2**: For high-throughput applications
3. **Jumbo Frames**: Configure MTU 9000 for storage networks
4. **CPU Pinning**: Combine with CPU manager for deterministic performance

### Operations

1. **Monitoring**: Export Multus metrics to Prometheus
2. **Logging**: Centralize CNI logs
3. **Documentation**: Maintain network topology diagrams
4. **Testing**: Regular connectivity and failover tests

## Monitoring Multus

### Prometheus Metrics

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: multus-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: multus
  endpoints:
  - port: metrics
    interval: 30s
```

### Key Metrics to Monitor

| Metric | Description |
|--------|-------------|
| `multus_cni_operations_total` | Total CNI operations |
| `multus_cni_operation_duration_seconds` | Operation latency |
| `multus_cni_errors_total` | Error count by type |
| `multus_network_attachments_total` | Active network attachments |

