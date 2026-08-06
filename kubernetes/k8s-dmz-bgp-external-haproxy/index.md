---
title: Kubernetes DMZ Ingress with HAProxy and BGP: External Mode Without Cilium External Workload
url: https://devopstales.github.io/kubernetes/k8s-dmz-bgp-external-haproxy/
date: 2026-02-26
keywords: kubernetes ingress, haproxy external mode, cilium bgp, dmz network, kubernetes security, bpf dataplane
---


Learn how to deploy HAProxy Ingress Controller on AlmaLinux in a DMZ network outside your Kubernetes cluster—without Cilium's deprecated external workload mode. This guide covers BGP peering with BIRD, Cilium's Pod CIDR export, firewalld configuration, and production-ready setup for secure ingress traffic isolation.

<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

> **Update for 2026:** Cilium has deprecated its external workload mode, requiring a new approach for placing HAProxy Ingress Controllers outside the Kubernetes cluster. This guide shows how to build a secure DMZ architecture using Cilium's BGP Pod CIDR export with standard BGP routing—no external workload mode required. All examples use AlmaLinux for the HAProxy DMZ node.

## Why the Change? Cilium External Workload Deprecation

Cilium's external workload mode allowed running Cilium agents on nodes outside the Kubernetes cluster. However, this feature has been deprecated in favor of simpler, more maintainable approaches.

The good news: **You don't need it anymore.** With Cilium's BGP control plane and Pod CIDR export, you can achieve the same DMZ architecture using standard BGP routing.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Internet                                       │
│                              ┌──────┐                                       │
│                              │ Users │                                       │
│                              └──┬───┘                                       │
│                                 │ HTTPS 443                                 │
└─────────────────────────────────┼───────────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼───────────────────────────────────────────┐
│                         DMZ Network 192.168.56.0/24                         │
│                                 │                                           │
│                          ┌──────▼───────┐                                   │
│                          │   HAProxy    │                                   │
│                          │ 192.168.56.15│                                   │
│                          └──────┬───────┘                                   │
│                                 │                                           │
│                    ┌────────────┼────────────┐                              │
│                    │            │            │                              │
│              BGP 3179    BGP 3179    BGP 3179                              │
│              HTTP/HTTPS  HTTP/HTTPS  HTTP/HTTPS                            │
│                │            │            │                                  │
└────────────────┼────────────┼────────────┼──────────────────────────────────┘
                 │            │            │
┌────────────────┼────────────┼────────────┼──────────────────────────────────┐
│           Internal K8s Network 192.168.10.0/24                              │
│                 │            │            │                                  │
│          ┌──────▼──────┐ ┌──▼────────┐ ┌─▼──────────┐                       │
│          │  Control    │ │  Worker 1 │ │  Worker 2  │                       │
│          │   Plane     │ │192.168.10.│ │192.168.10. │                       │
│          │192.168.10.  │ │21         │ │22         │                       │
│          │10-12        │ │Pod CIDR:  │ │Pod CIDR:   │                       │
│          │             │ │10.244.1.0 │ │10.244.2.0  │                       │
│          │             │ │/24        │ │/24         │                       │
│          │             │ │           │ │            │                       │
│          │             │ └───────────┘ └────────────┘                       │
│          │             │                                          ┌─────────┐
│          │             │                                          │Worker 3 │
│          │             │                                          │192.168. │
│          │             │                                          │10.23    │
│          │             │                                          │Pod CIDR:│
│          │             │                                          │10.244.3 │
│          │             │                                          │.0/24    │
│          │             │                                          └─────────┘
└──────────┴─────────────┴─────────────────────────────────────────────────────┘
```

**Key differences from the old approach:**

| Old (External Workload) | New (BGP Only) |
|------------------------|----------------|
| Cilium agent on HAProxy node | No Cilium on HAProxy node |
| Node in same network as K8s | HAProxy in separate DMZ network |
| VXLAN/Geneve overlay | Pure BGP routing |
| Complex setup | Simpler, standard BGP |

## Network Design

We'll create three network segments:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DMZ Zone                                       │
│                                                                             │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │  HAProxy                                                          │     │
│   │  eth0: 192.168.56.15/24                                           │     │
│   └───────────────────────────────────────────────────────────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         │ BGP Peering
         │ Port 3179
         │
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Internal Zone                                     │
│                                                                             │
│   ┌─────────────────────────────┐   ┌─────────────────────────────────┐     │
│   │  K8s Control Plane          │   │  K8s Workers                    │     │
│   │  eth0: 192.168.10.10-12/24  │   │  eth0: 192.168.10.21-23/24      │     │
│   └─────────────────────────────┘   └─────────────────────────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
         ▲
         │
         │ SSH/Kubernetes API
         │
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Management Zone                                     │
│                                                                             │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │  Admin Network                                                    │     │
│   │  192.168.100.0/24                                                 │     │
│   └───────────────────────────────────────────────────────────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Network Segments:**

| Network | CIDR | Purpose | Access |
|---------|------|---------|--------|
| DMZ | 192.168.56.0/24 | HAProxy external interface | Internet-facing |
| Internal | 192.168.10.0/24 | Kubernetes nodes | DMZ only, no direct internet |
| Pod Network | 10.244.0.0/16 | Pod CIDRs | Routed via BGP |
| Management | 192.168.100.0/24 | Admin access | Jump host only |

## Part 1: Configure Cilium BGP Control Plane

### Enable BGP in Cilium

First, enable the BGP control plane in your existing Cilium installation:

```bash
# Check current Cilium status
cilium status

# Create Helm values file for BGP
cat << EOF > cilium-bgp-values.yaml
bgpControlPlane:
  enabled: true

# Ensure BPF dataplane is enabled (default in Cilium 1.14+)
enableBpfMasquerade: true
routingMode: native
EOF

# Upgrade Cilium
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  -f cilium-bgp-values.yaml \
  --wait
```

Verify BGP CRDs are available:

```bash
kubectl api-resources | grep -i ciliumbgp

# Expected output:
# ciliumbgppeeringpolicies    bgpp    cilium.io/v2alpha1    false    CiliumBGPPeeringPolicy
# ciliumbgpclusterconfigs     bgpcc   cilium.io/v2alpha1    false    CiliumBGPClusterConfiguration
```

### Create BGP Cluster Configuration

Create `cilium-bgp-config.yaml`:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPClusterConfiguration
metadata:
  name: cluster-bgp-config
spec:
  localASN: 65001
  bgpPort: 3179  # Non-standard port (avoid privilege issues)
  loadBalancerIPs: []
  peerConfigs:
    - peerASN: 65001
      peerAddress: 192.168.56.15
      peerPort: 3179
      interface: eth0
      eBGPMultihopTTL: 0
      connectRetryTimeSeconds: 120
      holdTimeSeconds: 90
      keepAliveTimeSeconds: 30
      gracefulRestart:
        enabled: true
        restartTimeSeconds: 120
  addressPools:
    - name: pod-cidr-pool
      blocks:
        - 10.244.0.0/16
```

### Create BGP Peering Policy

Create `cilium-bgp-policy.yaml`:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: bgp-peering-policy
spec:
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/worker: ""
  virtualRouters:
    - localASN: 65001
      exportPodCIDR: true
      exportAddresses:
        - type: "podCIDR"
      neighbors:
        - peerAddress: '192.168.56.15/32'
          peerASN: 65001
          peerPort: 3179
          holdTime: 90s
          keepAliveInterval: 30s
          gracefulRestart:
            enabled: true
            restartTime: 120s
```

Apply the configurations:

```bash
kubectl apply -f cilium-bgp-config.yaml
kubectl apply -f cilium-bgp-policy.yaml

# Verify BGP sessions
cilium bgp peers

# Expected output:
# Node    Local AS   Peer AS   Peer Address     Session State   Uptime   Family          Received   Advertised
# w1      65001      65001     192.168.56.15    established     2m       ipv4/unicast    0          1
# w2      65001      65001     192.168.56.15    established     2m       ipv4/unicast    0          1
# w3      65001      65001     192.168.56.15    established     2m       ipv4/unicast    0          1
```

### Verify Pod CIDR Export

Check that worker nodes are advertising their Pod CIDRs:

```bash
# Get Pod CIDRs for each node
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'

# Example output:
# w1    10.244.1.0/24
# w2    10.244.2.0/24
# w3    10.244.3.0/24

# Check Cilium BGP advertisements
cilium bgp advertised-routes -n kube-system

# Should show Pod CIDRs being advertised to 192.168.56.15
```

## Part 2: Configure HAProxy DMZ Node

### System Requirements

This guide uses **AlmaLinux 9** for the HAProxy DMZ node. The configuration also works on RHEL 9, Rocky Linux 9, and other RHEL-compatible distributions.

```bash
# Verify OS version
cat /etc/redhat-release

# Expected output:
# AlmaLinux release 9.x (Turquoise Kodkod)

# Update system
sudo dnf update -y

# Install required tools
sudo dnf install -y vim wget curl net-tools telnet
```

### Network Configuration

The HAProxy node needs two network interfaces. On AlmaLinux 9, use NetworkManager:

```bash
# Check network interfaces
ip addr show

# Configure primary interface (DMZ - eth0)
sudo nmcli connection modify eth0 \
  ipv4.addresses 192.168.56.15/24 \
  ipv4.gateway 192.168.56.1 \
  ipv4.dns "8.8.8.8 1.1.1.1" \
  ipv4.method manual \
  connection.autoconnect yes

# Add route to Pod network (will be managed by BIRD)
sudo nmcli connection modify eth0 \
  +ipv4.routes "10.244.0.0/16 192.168.56.1"

# Configure secondary interface (Internal - eth1, optional)
sudo nmcli connection add \
  type ethernet \
  con-name eth1 \
  ifname eth1 \
  ipv4.addresses 192.168.10.100/24 \
  ipv4.method manual \
  ipv4.never-default yes \
  connection.autoconnect yes

# Apply changes
sudo nmcli connection up eth0
sudo nmcli connection up eth1

# Verify configuration
ip addr show
ip route show
```

### Install and Configure BIRD

BIRD will establish BGP peering with all Kubernetes workers:

```bash
# Install EPEL repository (required for BIRD)
sudo dnf install -y epel-release

# Install BIRD
sudo dnf install -y bird

# Backup default config
sudo cp /etc/bird/bird.conf /etc/bird/bird.conf.backup
```

Create `/etc/bird/bird.conf`:

```bash
cat << 'EOF' | sudo tee /etc/bird/bird.conf
# BIRD BGP Configuration for HAProxy DMZ Node
# Router ID should be unique in your network
router id 192.168.56.15;

# Logging
log syslog { debug };
log "/var/log/bird.log" all;

# BGP Graceful Restart
graceful restart wait time 300;

# ============================================
# BGP Peerings with Kubernetes Workers
# ============================================

protocol bgp w1 {
  local 192.168.56.15 as 65001;
  neighbor 192.168.10.21 port 3179 as 65001;

  # Import Pod CIDRs from K8s
  import all;

  # Don't export anything to K8s (we only need routes)
  export none;

  # Multi-hop required since peers are on different subnet
  multihop;

  # Graceful restart for failover
  graceful restart;

  # Hold time and keepalive
  hold time 90;
  keepalive time 30;
}

protocol bgp w2 {
  local 192.168.56.15 as 65001;
  neighbor 192.168.10.22 port 3179 as 65001;

  import all;
  export none;
  multihop;
  graceful restart;
  hold time 90;
  keepalive time 30;
}

protocol bgp w3 {
  local 192.168.56.15 as 65001;
  neighbor 192.168.10.23 port 3179 as 65001;

  import all;
  export none;
  multihop;
  graceful restart;
  hold time 90;
  keepalive time 30;
}

# ============================================
# Kernel Routing Table Integration
# ============================================

protocol kernel {
  scan time 10;        # Scan kernel routes every 10 seconds
  import all;          # Import kernel routes
  export all;          # Export BIRD routes to kernel
  persist;             # Keep routes after BIRD restart
  learn;               # Learn routes from kernel
  merge paths on;      # Enable ECMP for load balancing
}

# ============================================
# Device Discovery
# ============================================

protocol device {
  scan time 5;         # Scan interfaces every 5 seconds
}

# ============================================
# Route Filters (Optional but Recommended)
# ============================================

# Only accept Pod CIDR routes from K8s
filter k8s_routes {
  if net ~ [ 10.244.0.0/16+ ] then accept;
  else reject;
}

# Apply filter to BGP imports
# protocol bgp w1 { ... import filter k8s_routes; ... }
EOF
```

Start and verify BIRD:

```bash
# Enable and start BIRD
sudo systemctl enable bird
sudo systemctl start bird

# Check BIRD status
sudo systemctl status bird

# Verify BGP sessions
sudo birdc show protocols

# Expected output:
# name     proto    table    state  since       info
# w1       BGP      master   up     00:05:23    Established
# w2       BGP      master   up     00:05:25    Established
# w3       BGP      master   up     00:05:24    Established
# kernel1  Kernel   master   up     00:05:20
# device1  Device   master   up     00:05:20

# Check learned routes
sudo birdc show route

# Expected output:
# 10.244.1.0/24      via 192.168.10.21 on eth0 [w1 00:05:25] * (100) [i]
# 10.244.2.0/24      via 192.168.10.22 on eth0 [w2 00:05:27] * (100) [i]
# 10.244.3.0/24      via 192.168.10.23 on eth0 [w3 00:05:26] * (100) [i]

# Verify kernel routing table
ip route | grep 10.244

# Expected output:
# 10.244.1.0/24 via 192.168.10.21 dev eth0
# 10.244.2.0/24 via 192.168.10.22 dev eth0
# 10.244.3.0/24 via 192.168.10.23 dev eth0
```

### Install HAProxy

Install HAProxy version compatible with the Ingress Controller:

```bash
# Install EPEL repository (if not already installed)
sudo dnf install -y epel-release

# Install HAProxy from base repositories (AlmaLinux 9 has HAProxy 2.4+)
# Check available version
dnf info haproxy

# For HAProxy 2.8+, enable CRB repository and use external repo
sudo dnf config-manager --set-enabled crb

# Add HAProxy repository for version 2.8
curl https://haproxy.debian.net/bernat.debian.org.gpg | \
  sudo gpg --dearmor -o /etc/pki/rpm-gpg/RPM-GPG-KEY-haproxy

cat << EOF | sudo tee /etc/yum.repos.d/haproxy.repo
[haproxy]
name=HAProxy Repository
baseurl=http://haproxy.debian.net/bookworm-backports-2.8/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-haproxy
EOF

# Install HAProxy 2.8 (compatible with Ingress Controller 1.11+)
sudo dnf install -y haproxy

# Stop and disable system HAProxy service
sudo systemctl stop haproxy
sudo systemctl disable haproxy

# Allow HAProxy to bind to privileged ports
sudo setcap cap_net_bind_service=+ep /usr/sbin/haproxy

# Verify installation
haproxy -v
```

> **Note:** If you prefer to use the HAProxy version from AlmaLinux base repos (2.4+), you can skip the external repo and use `sudo dnf install -y haproxy`. Just ensure it's compatible with your Ingress Controller version.

### Install HAProxy Kubernetes Ingress Controller

Download and install the ingress controller binary:

```bash
# Download latest release
INGRESS_VERSION="1.11.3"
wget https://github.com/haproxytech/kubernetes-ingress/releases/download/v${INGRESS_VERSION}/haproxy-ingress-controller_${INGRESS_VERSION}_Linux_x86_64.tar.gz

# Extract and install
tar -xzvf haproxy-ingress-controller_${INGRESS_VERSION}_Linux_x86_64.tar.gz
sudo cp ./haproxy-ingress-controller /usr/local/bin/

# Verify installation
haproxy-ingress-controller --version
```

### Configure Kubernetes Access

Copy kubeconfig from control plane:

```bash
# On control plane node
sudo scp /etc/kubernetes/admin.conf haproxy-dmz:/root/.kube/config

# On HAProxy node
sudo mkdir -p /root/.kube
sudo chown root:root /root/.kube/config
chmod 600 /root/.kube/config

# Test connectivity
kubectl cluster-info
kubectl get nodes
```

Create the ConfigMap for HAProxy:

```bash
kubectl create configmap haproxy-kubernetes-ingress \
  --namespace default \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Create Systemd Service

Create `/etc/systemd/system/haproxy-ingress.service`:

```ini
[Unit]
Description=HAProxy Kubernetes Ingress Controller
Documentation=https://www.haproxy.com/
Requires=network-online.target
After=network-online.target bird.service
Wants=bird.service

[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/local/bin/haproxy-ingress-controller \
  --external \
  --default-ssl-certificate=ingress-system/default-cert \
  --configmap=default/haproxy-kubernetes-ingress \
  --program=/usr/sbin/haproxy \
  --disable-ipv6 \
  --ipv4-bind-address=0.0.0.0 \
  --http-bind-port=80 \
  --https-bind-port=443 \
  --ingress.class=ingress-public \
  --kubeconfig=/root/.kube/config \
  --watch-gateway=false \
  --update-status=false
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGTERM
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log /var/lib/haproxy

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable haproxy-ingress
sudo systemctl start haproxy-ingress

# Check status
sudo systemctl status haproxy-ingress

# View logs
journalctl -u haproxy-ingress -f
```

## Part 3: Firewall and Security Configuration

### HAProxy Node Firewall

Configure firewalld on the HAProxy node (AlmaLinux default firewall):

```bash
# Enable and start firewalld (should be running by default)
sudo systemctl enable firewalld
sudo systemctl start firewalld

# Set default zone to drop
sudo firewall-cmd --set-default-zone=drop

# Allow SSH from management network only
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.100.0/24" service name="ssh" accept'

# Allow HTTP/HTTPS from internet
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https

# Allow BGP from K8s workers only
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.21/32" port port="3179" protocol="tcp" accept'
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.22/32" port port="3179" protocol="tcp" accept'
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.23/32" port port="3179" protocol="tcp" accept

# Allow established and related connections
sudo firewall-cmd --permanent --add-rich-rule='rule protocol value="tcp" accept'

# Reload firewalld to apply changes
sudo firewall-cmd --reload

# Verify configuration
sudo firewall-cmd --list-all
sudo firewall-cmd --list-rich-rules

# Expected output:
# public (active)
#   target: DROP
#   icmp-block-inversion: no
#   interfaces: eth0 eth1
#   sources:
#   services: http https
#   ports:
#   protocols:
#   forward: no
#   masquerade: no
#   forward-ports:
#   source-ports:
#   icmp-blocks:
#   rich rules:
#       rule family="ipv4" source address="192.168.100.0/24" service name="ssh" accept
#       rule family="ipv4" source address="192.168.10.21/32" port port="3179" protocol="tcp" accept
#       rule family="ipv4" source address="192.168.10.22/32" port port="3179" protocol="tcp" accept
#       rule family="ipv4" source address="192.168.10.23/32" port port="3179" protocol="tcp" accept
#       rule protocol value="tcp" accept
```

### Kubernetes Worker Firewall

On each Kubernetes worker node (assuming AlmaLinux/Rocky/RHEL):

```bash
# Enable and start firewalld
sudo systemctl enable firewalld
sudo systemctl start firewalld

# Set default zone to drop
sudo firewall-cmd --set-default-zone=drop

# Allow SSH from management network only
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.100.0/24" service name="ssh" accept'

# Allow BGP from HAProxy only
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.15/32" port port="3179" protocol="tcp" accept'

# Allow Kubernetes API from internal network
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.0/24" port port="6443" protocol="tcp" accept'

# Allow Cilium/CNI traffic (Pod network)
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.244.0.0/16" accept'

# Allow established and related connections
sudo firewall-cmd --permanent --add-rich-rule='rule protocol value="tcp" accept'

# Reload firewalld to apply changes
sudo firewall-cmd --reload

# Verify configuration
sudo firewall-cmd --list-all
```

## Part 4: Deploy Test Application

Create `demo-app.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  annotations:
    ingressclass.kubernetes.io/is-default-class: "false"
  name: ingress-public
spec:
  controller: haproxy.org/ingress-controller/ingress-public
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: echo-server
  name: echo-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: echo-server
  template:
    metadata:
      labels:
        app: echo-server
    spec:
      containers:
      - name: echo-server
        image: jmalloc/echo-server:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: echo-server
              topologyKey: kubernetes.io/hostname
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: echo-server
  name: echo-server
spec:
  selector:
    app: echo-server
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-server-ingress
  annotations:
    haproxy.org/balance-algorithm: roundrobin
    haproxy.org/ssl-redirect: "true"
spec:
  ingressClassName: ingress-public
  rules:
  - host: "echo.example.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: echo-server
            port:
              number: 80
```

Deploy the application:

```bash
kubectl apply -f demo-app.yaml

# Verify deployment
kubectl get pods -l app=echo-server
kubectl get svc echo-server
kubectl get ingress echo-server-ingress
```

## Part 5: Testing and Verification

### Test BGP Connectivity

From HAProxy node:

```bash
# Check BGP sessions
sudo birdc show protocols

# Verify routes
sudo birdc show route

# Test connectivity to Pod network
ping -c 3 10.244.1.10  # Replace with actual pod IP
```

### Test Ingress Access

From external network:

```bash
# Add hosts entry for testing
echo "192.168.56.15 echo.example.com" | sudo tee -a /etc/hosts

# Test HTTP access
curl -I http://echo.example.com

# Test HTTPS access (with SSL redirect)
curl -I https://echo.example.com

# Test with verbose output
curl -v http://echo.example.com
```

### Test High Availability

```bash
# Check which worker is handling traffic
kubectl get pods -l app=echo-server -o wide

# Simulate worker failure
kubectl cordon w1
kubectl drain w1 --ignore-daemonsets --delete-emptydir-data

# Verify traffic shifts to remaining workers
curl http://echo.example.com

# Restore worker
kubectl uncordon w1

# Verify BGP session re-establishes
sudo birdc show protocols
```

### Monitor BGP Sessions

Create a monitoring script `/usr/local/bin/check-bgp.sh`:

```bash
#!/bin/bash

# Check BGP session status
STATUS=$(sudo birdc show protocols | grep -E "^w[0-9]" | awk '{print $4}')

FAILED=$(echo "$STATUS" | grep -v "up" | wc -l)

if [ $FAILED -gt 0 ]; then
    echo "CRITICAL: $FAILED BGP session(s) down"
    exit 2
elif [ $FAILED -eq 0 ]; then
    echo "OK: All BGP sessions established"
    exit 0
else
    echo "WARNING: Some BGP sessions unstable"
    exit 1
fi
```

## Troubleshooting

### BGP Session Not Establishing

```bash
# On HAProxy node (AlmaLinux)
sudo birdc show protocols all
sudo tail -f /var/log/bird.log

# Check firewalld rules
sudo firewall-cmd --list-rich-rules | grep 3179

# Temporarily allow all for testing
sudo firewall-cmd --permanent --add-rich-rule='rule protocol value="tcp" accept'
sudo firewall-cmd --reload

# On Kubernetes worker
cilium bgp peers
kubectl logs -n kube-system -l k8s-app=cilium | grep -i bgp

# Check firewall rules on worker
sudo firewall-cmd --list-rich-rules | grep 3179

# Verify network connectivity
telnet 192.168.10.21 3179
```

### Routes Not Appearing

```bash
# Check if Pod CIDRs are exported
cilium bgp advertised-routes -n kube-system

# Verify BIRD is importing
sudo birdc show route all

# Check kernel routing table
ip route show table all
```

### HAProxy Not Receiving Traffic

```bash
# Check ingress controller logs
journalctl -u haproxy-ingress -f

# Verify HAProxy configuration
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

# Check ingress resource
kubectl describe ingress echo-server-ingress
kubectl get endpoints echo-server
```

## Conclusion

This architecture provides a secure, production-ready ingress solution:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                               Benefits                                      │
│                                                                             │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐         │
│   │ ✅ DMZ Isolation │  │ ✅ No Cilium on  │  │ ✅ Standard BGP  │         │
│   │                  │  │    HAProxy       │  │    Routing       │         │
│   └──────────────────┘  └──────────────────┘  └──────────────────┘         │
│                                                                             │
│   ┌──────────────────┐  ┌──────────────────┐                               │
│   │ ✅ Easy to       │  │ ✅ High          │                               │
│   │    Maintain      │  │    Availability  │                               │
│   └──────────────────┘  └──────────────────┘                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                            Security Zones                                   │
│                                                                             │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐                │
│   │  Internet   │─────>│ DMZ:        │─────>│ Internal:   │                │
│   │             │ Only │ HAProxy     │ BGP  │ K8s         │                │
│   │             │ 80/44│             │ 3179 │             │                │
│   │             │      │             │ +HTTP│             │                │
│   └─────────────┘      └─────────────┘      └─────────────┘                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key advantages over the deprecated external workload mode:**

1. **Simpler architecture** - No Cilium agent on HAProxy node
2. **Better isolation** - True DMZ network separation
3. **Standard protocols** - Pure BGP, no vendor-specific extensions
4. **Easier troubleshooting** - Standard BGP tools (birdc, show commands)
5. **Flexible deployment** - HAProxy can be anywhere with BGP connectivity

The trade-off is slightly more complex BGP configuration, but the security and maintainability benefits are worth it for production environments.

