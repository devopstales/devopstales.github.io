---
title: Kubernetes Secure Install
url: https://devopstales.github.io/kubernetes/k8s-secure-install/
date: 2024-01-20
keywords: Kubernetes Hardening Guide, kubernetes deployment, Kubernetes Secuity, Best Practices, CRI-O, Cilium, seccomp, selinux, User namespace, CISA
---


In this post I will show you how to install a Kubernetes cluster in a secure way with.

<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

I will start wit an Almalinux 9. Wirst I will install the nececary packages end enable repositories. TheI will upgrade the system to the latest package versions. 

```bash
dnf config-manager --set-enabled crb
dnf install -y epel-release
dnf install -y nano wget
dnf upgrade -y
```

### Selinux and Firewall Config

```bash
systemctl enable firewalld
systemctl start firewalld

# Check selinux enabled
sestatus

# Resoult
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```

Now that we enabéed the firewall we will create the nececary firewall rules for the Kubernetes components.

```bash
sudo firewall-cmd --add-port=9345/tcp --permanent
sudo firewall-cmd --add-port=6443/tcp --permanent
sudo firewall-cmd --add-port=10250/tcp --permanent
sudo firewall-cmd --add-port=2379/tcp --permanent
sudo firewall-cmd --add-port=2380/tcp --permanent
sudo firewall-cmd --add-port=30000-32767/tcp --permanent
# Used for the Monitoring
sudo firewall-cmd --add-port=9796/tcp --permanent
sudo firewall-cmd --add-port=19090/tcp --permanent
sudo firewall-cmd --add-port=6942/tcp --permanent
sudo firewall-cmd --add-port=9091/tcp --permanent
### CNI specific ports Cilium
# 4244/TCP is required when the Hubble Relay is enabled and therefore needs to connect to all agents to collect the flows
sudo firewall-cmd --add-port=4244/tcp --permanent
# Cilium healthcheck related permits:
sudo firewall-cmd --add-port=4240/tcp --permanent
sudo firewall-cmd --remove-icmp-block=echo-request --permanent
sudo firewall-cmd --remove-icmp-block=echo-reply --permanent
# Since we are using Cilium with GENEVE as overlay, we need the following port too:
sudo firewall-cmd --add-port=6081/udp --permanent
### Ingress Controller specific ports
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --add-port=443/tcp --permanent
### To get DNS resolution working, simply enable Masquerading.
sudo firewall-cmd --zone=public  --add-masquerade --permanent

### Finally apply all the firewall changes
sudo firewall-cmd --reload
```

Verification:

```bash
sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eno1
  sources: 
  services: cockpit dhcpv6-client ssh wireguard
  ports: 9345/tcp 6443/tcp 10250/tcp 2379/tcp 2380/tcp 30000-32767/tcp 4240/tcp 6081/udp 80/tcp 443/tcp 4244/tcp 9796/tcp 19090/tcp 6942/tcp 9091/tcp
  protocols: 
  masquerade: yes
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

### Linux Configurations

Now we will enable cgroup V2 for better performance and security.

```bash
sudo dnf install -y grubby
sudo grubby \
  --update-kernel=ALL \
  --args="systemd.unified_cgroup_hierarchy=1"

cat << EOF >> /etc/systemd/system.conf
DefaultCPUAccounting=yes
DefaultIOAccounting=yes
DefaultIPAccounting=yes
DefaultBlockIOAccounting=yes
EOF

init 6
```

```bash
# check for type cgroup2
$ mount -l|grep cgroup
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,seclabel,nsdelegate)

# check for cpu controller
$ cat /sys/fs/cgroup/cgroup.subtree_control
cpu io memory pids
```

```bash
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
#
# protectKernelDefaults
#
kernel.keys.root_maxbytes           = 25000000
kernel.keys.root_maxkeys            = 1000000
kernel.panic                        = 10
kernel.panic_on_oops                = 1
vm.overcommit_memory                = 1
vm.panic_on_oom                     = 0
EOF

sysctl --system
```

Ensure the eBFP filesystem is mounted (which should already be the case on RHEL 8.3):

```bash
mount | grep /sys/fs/bpf
# if present should output, e.g. "none on /sys/fs/bpf type bpf"...
```

If that's not the case, mount it using the commands down here:

```bash
sudo mount bpffs -t bpf /sys/fs/bpf
sudo bash -c 'cat <<EOF >> /etc/fstab
none /sys/fs/bpf bpf rw,relatime 0 0
EOF'
```

### CRI-O

```bash
KUBERNETES_VERSION=v1.32
CRIO_VERSION=v1.32

cat <<EOF | tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/rpm/repodata/repomd.xml.key
EOF

yum install -y cri-o container-selinux fuse-overlayfs conmon
```

```bash
# Configure User anmespacing in CRI-O
mkdir /etc/crio/crio.conf.d/

sed -i 's|^cgroup_manager|#cgroup_manager|' /etc/crio/crio.conf.d/10-crio.conf
sed -i 's|^conmon_cgroup|#conmon_cgroup|' /etc/crio/crio.conf.d/10-crio.conf

cat <<EOF > /etc/crio/crio.conf.d/01-crio-base.conf
[crio]
storage_driver = "overlay"
storage_option = ["overlay.mount_program=/usr/bin/fuse-overlayfs"]

[crio.runtime]
selinux = true
cgroup_manager = "cgroupfs"
conmon_cgroup = 'pod'
EOF

cat <<EOF > /etc/crio/crio.conf.d/02-userns-workload.conf
[crio.runtime.workloads.userns]
activation_annotation = "io.kubernetes.cri-o.userns-mode"
allowed_annotations = ["io.kubernetes.cri-o.userns-mode"]
EOF
```

The **CRI-O** will run the containers with the `containers` user so I need to create `/etc/subuid` and `/etc/subgid` on nodes.

> SubUID/GIDs are a range of user/group IDs that a user is allowed to use.

```bash
echo "containers:200000:268435456" >> /etc/subuid
echo "containers:200000:268435456" >> /etc/subgid
```

### Install gVisor

gVisor provides an additional layer of isolation between running applications and the host operating system. It includes an Open Container Initiative (OCI) runtime called `runsc` that makes it easy to work with existing container tooling. The `runsc` runtime integrates with Docker, CRI-O and Kubernetes, making it simple to run sandboxed containers.

```bash
cat <<EOF > ~/gvisor.sh
#!/bash
(
  set -e
  ARCH=$(uname -m)
  URL=https://storage.googleapis.com/gvisor/releases/release/latest/${ARCH}
  wget ${URL}/runsc ${URL}/runsc.sha512 \
    ${URL}/containerd-shim-runsc-v1 ${URL}/containerd-shim-runsc-v1.sha512
  sha512sum -c runsc.sha512 \
    -c containerd-shim-runsc-v1.sha512
  rm -f *.sha512
  chmod a+rx runsc containerd-shim-runsc-v1
  sudo mv runsc containerd-shim-runsc-v1 /usr/local/bin
)
EOF
```

```bash
bash gvisor.sh
...
runsc: OK
containerd-shim-runsc-v1: OK
```

Now I need to configure `CRI-O` to use `runsc` as low-level runetime egine.

```bash
mkdir /etc/crio/crio.conf.d/
cat <<EOF > /etc/crio/crio.conf.d/99-gvisor
# Path to the gVisor runtime binary that uses runsc
[crio.runtime.runtimes.runsc]
runtime_path = "/usr/local/bin/runsc"
EOF

systemctl enable crio
systemctl restart crio
systemctl status crio
```

### Kubeadm install

```bash
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
EOF

CRIP_VERSION=$(crio --version | grep "^Version" | awk '{print $2}')
yum install kubelet-$CRIP_VERSION kubeadm-$CRIP_VERSION kubectl-$CRIP_VERSION cri-tools iproute-tc -y

echo "exclude=kubelet, kubectl, kubeadm, cri-o" >> /etc/yum.conf
```

### Kubeadm init config

```bash
nano 010-kubeadm-conf-1-32-0.yaml
```

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
bootstrapTokens:
- token: "c2t0rj.cofbfnwwrb387890"
  ttl: "48h"
  usages:
  - signing
  - authentication
localAPIEndpoint:
  # local ip and port
  advertiseAddress: 192.168.100.10
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/crio/crio.sock
  imagePullPolicy: IfNotPresent
  taints: []
```

### Install Kubernetes

```bash
kubeadm config images pull --config 010-kubeadm-conf-1-32-2.yaml

systemctl enable kubelet.service

kubeadm init --skip-phases=addon/kube-proxy --config 010-kubeadm-conf-1-32-2.yaml
```

### Post Install

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get csr --all-namespaces
kubectl get csr -oname | xargs kubectl certificate approve
kubectl apply -f 012-k8s-clusterrole.yaml

yum install -y https://harbottle.gitlab.io/harbottle-main/7/x86_64/harbottle-main-release.rpm
yum install -y kubectx

curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
echo 'PATH=$PATH:/usr/local/bin' >> /etc/profile
export PATH=$PATH:/usr/local/bin
```

### Install cilium as CNI

Generate Clilium configuration:

```yaml
cat <<EOF > 031-cilium-helm-values.yaml
# Set kubeProxyReplacement to "strict" in order to prevent CVE-2020-8554 and fully remove kube-proxy.
# See https://cilium.io/blog/2020/12/11/kube-proxy-free-cve-mitigation for more information.
kubeProxyReplacement: "strict"

k8sServiceHost: 192.168.56.12
k8sServicePort: 6443
rollOutCiliumPods: true
priorityClassName: system-cluster-critical

ipv4:
  enabled: true
ipv6:
  enabled: false

bpf:
  masquerade: true

encryption:
  type: wireguard
  enabled: false
  nodeEncryption: false

# L7 policy
loadBalancer:
  l7:
    backend: envoy
envoy:
  enabled: true
  prometheus:
    enabled: true
    serviceMonitor:
      enabled: false

# L2 LoadBalancer service
l2announcements:
  enabled: true

# Api gateway
gatewayAPI:
  enabled: false

# Ingress controller
ingressController:
  enabled: false
  loadbalancerMode: shared

# mTLS
authentication:
  mode: required
  mutual:
    spire:
      enabled: false
      install:
        enabled: false
        server:
          dataStorage:
            enabled: false

endpointStatus:
  enabled: true
  status: policy

dashboards:
  enabled: false
  namespace: "monitoring-system"
  annotations:
    grafana_folder: "cilium"

hubble:
  enabled: true
  metrics:
    enableOpenMetrics: true
    enabled:
    - dns:query;ignoreAAAA
    - drop
    - tcp
    - flow:sourceContext=workload-name|reserved-identity;destinationContext=workload-name|reserved-identity
    - port-distribution
    - icmp
    - kafka:labelsContext=source_namespace,source_workload,destination_namespace,destination_workload,traffic_direction;sourceContext=workload-name|reserved-identity;destinationContext=workload-name|reserved-identity
    - policy:sourceContext=app|workload-name|pod|reserved-identity;destinationContext=app|workload-name|pod|dns|reserved-identity;labelsContext=source_namespace,destination_namespace
    - httpV2:exemplars=true;labelsContext=source_ip,source_namespace,source_workload,destination_ip,destination_namespace,destination_workload,traffic_direction
    serviceMonitor:
      enabled: false
    dashboards:
      enabled: false
      namespace: "monitoring-system"
      annotations:
        grafana_folder: "cilium"

  ui:
    enabled: true
    replicas: 1
    ingress:
      enabled: true
      hosts:
        - hubble.k8s.intra
      annotations:
        kubernetes.io/ingress.class: nginx
        cert-manager.io/cluster-issuer: ca-issuer
      tls:
      - secretName: hubble-ingress-tls
        hosts:
        - hubble.k8s.intra
    tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
    backend:
      resources:
        limits:
          cpu: 60m
          memory: 300Mi
        requests:
          cpu: 20m
          memory: 64Mi
    frontend:
      resources:
        limits:
          cpu: 1000m
          memory: 1024M
        requests:
          cpu: 100m
          memory: 64Mi
    proxy:
      resources:
        limits:
          cpu: 1000m
          memory: 1024M
        requests:
          cpu: 100m
          memory: 64Mi

  relay:
    enabled: true
    tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
    resources:
      limits:
        cpu: 100m
        memory: 500Mi
    prometheus:
      enabled: true
      serviceMonitor:
        enabled: false

operator:
  replicas: 1
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 100m
      memory: 128Mi
  prometheus:
    enabled: true
    serviceMonitor:
      enabled: false
  dashboards:
    enabled: false
    namespace: "monitoring-system"
    annotations:
      grafana_folder: "cilium"

ipam:
  mode: "cluster-pool"
  operator:
    clusterPoolIPv4PodCIDRList: "10.43.0.0/16"
    clusterPoolIPv4MaskSize: 24
    clusterPoolIPv6PodCIDRList: "fd00::/104"
    clusterPoolIPv6MaskSize: 120

resources:
  limits:
    cpu: 4000m
    memory: 4Gi
  requests:
    cpu: 100m
    memory: 512Mi

prometheus:
  enabled: true
  # Default port value (9090) needs to be changed since the RHEL cockpit also listens on this port.
  port: 19090
  # Configure this serviceMonitor section AFTER Rancher Monitoring is enabled!
  serviceMonitor:
    enabled: false
EOF
```

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-

helm repo add cilium https://helm.cilium.io/
helm upgrade --install cilium cilium/cilium \
  --namespace kube-system \
  -f 031-cilium-helm-values.yaml

kubectl get pods -A
```

### Harden Kubernetes

There is an opensource tool theat tests CISA's best best practices on your clsuter. We vill use this to test the resoults. 

```bash
# kube-bench
# https://github.com/aquasecurity/kube-bench/releases/
yum install -y https://github.com/aquasecurity/kube-bench/releases/download/v0.6.5/kube-bench_0.6.5_linux_amd64.rpm
```

```bash
useradd -r -c "etcd user" -s /sbin/nologin -M etcd
chown etcd:etcd /var/lib/etcd
chmod 700 /var/lib/etcd

# kube-bench
kube-bench
kube-bench | grep "\[FAIL\]"
```

### join nodes

Firs we need to get the join command from the master:

```bash
# on master1
kubeadm token create --print-join-command
kubeadm join 192.168.56.12:6443 --token c2t0rj.cofbfnwwrb387890 \
 --discovery-token-ca-cert-hash sha256:a52f4c16a6ce9ef72e3d6172611d17d9752dfb1c3870cf7c8ad4ce3bcb97547e
```

If the next node is a worke we can just use the command what we get. If a next node is a master we need to generate a certificate-key. You need a separate certificate-key for every new master.

```bash
# on master1
## generate cert key
kubeadm certs certificate-key
29ab8a6013od73s8d3g4ba3a3b24679693e98acd796356eeb47df098c47f2773

## store cert key in secret
kubeadm init phase upload-certs --upload-certs --certificate-key=29ab8a6013od73s8d3g4ba3a3b24679693e98acd796356eeb47df098c47f2773
```

```bash
# on master2
kubeadm join 192.168.56.12:6443 --token c2t0rj.cofbfnwwrb387890 \
--discovery-token-ca-cert-hash sha256:a52f4c16a6ce9ef72e3d6172611d17d9752dfb1c3870cf7c8ad4ce3bcb97547e \
--control-plane --certificate-key 29ab8a6013od73s8d3g4ba3a3b24679693e98acd796356eeb47df098c47f2773
```

```bash
# on master3
kubeadm join 192.168.56.12:6443 --token c2t0rj.cofbfnwwrb387890 \
--discovery-token-ca-cert-hash sha256:a52f4c16a6ce9ef72e3d6172611d17d9752dfb1c3870cf7c8ad4ce3bcb97547e \
--control-plane --certificate-key 29ab8a6013od73s8d3g4ba3a3b24679693e98acd796356eeb47df098c47f2773
```

In the end withevery new node we need to approve the certificate requests for the node.

```bash
kubectl get csr -oname | xargs kubectl certificate approve

useradd -r -c "etcd user" -s /sbin/nologin -M etcd
chown etcd:etcd /var/lib/etcd
chmod 700 /var/lib/etcd
```

