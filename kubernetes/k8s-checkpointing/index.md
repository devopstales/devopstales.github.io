---
title: Kubernetes Container Checkpoints
url: https://devopstales.github.io/kubernetes/k8s-checkpointing/
date: 2024-01-14
keywords: kubernetes deployment, CRI-O, Checkpoints
---


In this post I will show you how to use Container Checkpoints.

<!--more-->

{{< content "/filedir/kubernetes.html" >}}

### What is Container Checkpoints?
Container Checkpointing feature allows you to checkpoint a running container. This means that you can save the current container state, without losing any information about the running processes or the data stored in it. Than resume it later in the future. This Feature is only available from Kubernetes 1.25. To implement Kubernetes checkpointing, you’ll need to use a container runtime that supports CRIU (Checkpoint/Restore in Userspace). 


> I will use Almalinux 8 as my OS.

### Install Kernel >5.13

Linux 5.13 added `PTRACE_GET_RSEQ_CONFIGURATION` feature that is neaded for Container Checkpoints feature.

```bash
sudo rpm --import https://repo.almalinux.org/almalinux/RPM-GPG-KEY-AlmaLinux
sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
sudo dnf install https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm -y

yum install epel-release
yum list available --disablerepo='*' --enablerepo=elrepo-kernel
```

### Install CRI-O
```bash
VERSION=1.28
sudo curl -L -o /etc/yum.repos.d/devel_kubic_libcontainers_stable.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/CentOS_8/devel:kubic:libcontainers:stable.repo
sudo curl -L -o /etc/yum.repos.d/devel_kubic_libcontainers_stable_cri-o_${VERSION}.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:${VERSION}/CentOS_8/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo

yum install cri-o nano wget net-tools bridge-utils dnsutils
```

### Configure
```bash
cat <<'EOF' | sudo tee /etc/modules-load.d/crio.conf > /dev/null
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
```

```bash
free -h
swapoff -a
swapoff -a
sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab
free -h
```

```bash
# Enable CRIU support in /etc/crio/crio.conf (enable_criu_support = true)
sed -i -e 's/# enable_criu_support = false/enable_criu_support = true/g' /etc/crio/crio.conf
sed -i -e 's/# drop_infra_ctr = true/drop_infra_ctr = false/g' /etc/crio/crio.conf
```

### Install kubernets
```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

CRIP_VERSION=$(crio --version | grep "^Version" | awk '{print $2}')
yum install kubelet-$CRIP_VERSION kubeadm-$CRIP_VERSION kubectl-$CRIP_VERSION cri-tools iproute-tc -y

echo "exclude=kubelet, kubectl, kubeadm" >> /etc/yum.conf
```

```yaml
# find the node IP
IP=172.17.9.10

cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: $IP
  bindPort: 6443
nodeRegistration:  # Only for CRI-O
  criSocket: "unix:///var/run/crio/crio.sock"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
featureGates:
  ContainerCheckpoint: true
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v$CRIP_VERSION
apiServer:
  extraArgs:
    feature-gates: "ContainerCheckpoint=true"
controllerManager:
  extraArgs:
    feature-gates: "ContainerCheckpoint=true"
scheduler:
  extraArgs:
    feature-gates: "ContainerCheckpoint=true"
networking:
  podSubnet: 10.244.0.0/16
EOF
```

```bash
systemctl enable kubelet.service
systemctl enable --now crio
kubeadm config images pull --cri-socket=unix:///var/run/crio/crio.sock --kubernetes-version=$CRIP_VERSION
kubeadm init --config=kubeadm-config.yaml --upload-certs | tee kubeadm-init.out

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


crictl info
kubectl get node -o wide
kubectl get po --all-namespaces
```

### Inincialize network
```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml
```

### Install tools
```bash
yum install git -y

sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/sbin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/sbin/kubens
```

```bash
kubens kube-system
kubectl describe po coredns-5dd5756b68-bztrx
```

If you have the fallowing error: `failed to delegate add: failed to set bridge addr: "cni0" already has an IP address different from 10.244.0.1/24`

```bash
ifconfig cni0 down    
ifconfig flannel.1 down    
ip link delete cni0
ip link delete flannel.1
```

### Demo for creating checkpoint:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-

kubens default
kubectl run nginx --image=nginx --restart=Never

curl -X POST "https://localhost:10250/checkpoint/<namespace>/<podId>/<container>"


curl -sk -X POST  "https://localhost:10250/checkpoint/default/nginx/nginx" \
  --key /etc/kubernetes/pki/apiserver-kubelet-client.key \
  --cacert /etc/kubernetes/pki/ca.crt \
  --cert /etc/kubernetes/pki/apiserver-kubelet-client.crt

# Response:
# {"items":["/var/lib/kubelet/checkpoints/checkpoint-nginx_default-nginx-2024-01-25T10:01:43Z.tar"]}

# Check the directory:
ls -l /var/lib/kubelet/checkpoints/


````

This request created an archive in `/var/lib/kubelet/checkpoints/checkpoint-<pod>_<namespace>-<container>-<timestamp>.tar`. 

### Analyzing

We now have a checkpointed container archive, so let's take a look at what's inside: 

```bash
cd /var/lib/kubelet/checkpoints/

cp checkpoint-nginx_default-nginx-2024-01-25T10:01:43Z.tar nginx.tar

tar --exclude="*/*" -tf nginx.tar

# Response:
stats-dump
dump.log
checkpoint/
config.dump
spec.dump
bind.mounts
rootfs-diff.tar
io.kubernetes.cri-o.LogPath

# Extract:
tar -xf nginx.tar

ll checkpoint
total 4816
-rw-r--r--. 1 root root    8305 Jan 25 10:01 cgroup.img
-rw-r--r--. 1 root root    1993 Jan 25 10:01 core-1.img
-rw-r--r--. 1 root root    2074 Jan 25 10:01 core-28.img
-rw-r--r--. 1 root root    2072 Jan 25 10:01 core-29.img
-rw-------. 1 root root      43 Jan 25 10:01 descriptors.json
-rw-r--r--. 1 root root     398 Jan 25 10:01 fdinfo-2.img
-rw-r--r--. 1 root root     324 Jan 25 10:01 fdinfo-3.img
-rw-r--r--. 1 root root     324 Jan 25 10:01 fdinfo-4.img
-rw-r--r--. 1 root root    2338 Jan 25 10:01 files.img
-rw-r--r--. 1 root root      18 Jan 25 10:01 fs-1.img
-rw-r--r--. 1 root root      18 Jan 25 10:01 fs-28.img
-rw-r--r--. 1 root root      18 Jan 25 10:01 fs-29.img
-rw-r--r--. 1 root root      36 Jan 25 10:01 ids-1.img
-rw-r--r--. 1 root root      36 Jan 25 10:01 ids-28.img
-rw-r--r--. 1 root root      36 Jan 25 10:01 ids-29.img
-rw-r--r--. 1 root root      46 Jan 25 10:01 inventory.img
-rw-r--r--. 1 root root      82 Jan 25 10:01 ipcns-var-11.img
-rw-r--r--. 1 root root      37 Jan 25 10:01 memfd.img
-rw-r--r--. 1 root root    1972 Jan 25 10:01 mm-1.img
-rw-r--r--. 1 root root    2083 Jan 25 10:01 mm-28.img
-rw-r--r--. 1 root root    2083 Jan 25 10:01 mm-29.img
-rw-r--r--. 1 root root    7831 Jan 25 10:01 mountpoints-13.img
-rw-r--r--. 1 root root      26 Jan 25 10:01 netns-10.img
-rw-r--r--. 1 root root     398 Jan 25 10:01 pagemap-1.img
-rw-r--r--. 1 root root     430 Jan 25 10:01 pagemap-28.img
-rw-r--r--. 1 root root     430 Jan 25 10:01 pagemap-29.img
-rw-r--r--. 1 root root      24 Jan 25 10:01 pagemap-shmem-1024.img
-rw-r--r--. 1 root root    4096 Jan 25 10:01 pages-1.img
-rw-r--r--. 1 root root 1286144 Jan 25 10:01 pages-2.img
-rw-r--r--. 1 root root 1740800 Jan 25 10:01 pages-3.img
-rw-r--r--. 1 root root 1740800 Jan 25 10:01 pages-4.img
-rw-r--r--. 1 root root      50 Jan 25 10:01 pstree.img
-rw-r--r--. 1 root root      12 Jan 25 10:01 seccomp.img
-rw-r--r--. 1 root root      32 Jan 25 10:01 timens-0.img
-rw-r--r--. 1 root root     408 Jan 25 10:01 tmpfs-dev-195.tar.gz.img
-rw-r--r--. 1 root root     367 Jan 25 10:01 tmpfs-dev-197.tar.gz.img
-rw-r--r--. 1 root root      98 Jan 25 10:01 tmpfs-dev-198.tar.gz.img
-rw-r--r--. 1 root root      98 Jan 25 10:01 tmpfs-dev-199.tar.gz.img
-rw-r--r--. 1 root root      98 Jan 25 10:01 tmpfs-dev-200.tar.gz.img
-rw-r--r--. 1 root root      27 Jan 25 10:01 utsns-12.img

cat config.dump
{
  "id": "0e740e048ccfb41adabad2bb4153781c5b8a65f4c50fc7eddc2148802fd994da",
  "name": "k8s_nginx_nginx_default_0ff70d03-6f65-4df7-a996-3eb313b9f63f_0",
  "rootfsImage": "docker.io/library/nginx@sha256:161ef4b1bf7effb350a2a9625cb2b59f69d54ec6059a8a155a1438d0439c593c",
  "rootfsImageRef": "a8758716bb6aa4d90071160d27028fe4eaee7ce8166221a97d30440c8eac2be6",
  "rootfsImageName": "docker.io/library/nginx:latest",
  "runtime": "runc",
  "createdTime": "2024-01-25T10:01:26.916550745Z",
  "checkpointedTime": "2024-01-25T10:01:43.05793532Z",
  "restoredTime": "0001-01-01T00:00:00Z",
  "restored": false
}

tar -tf rootfs-diff.tar
var/cache/nginx/scgi_temp/
var/cache/nginx/uwsgi_temp/
var/cache/nginx/client_temp/
var/cache/nginx/fastcgi_temp/
var/cache/nginx/proxy_temp/
run/nginx.pid
etc/mtab
```

### Restoring

To restore the previously check-pointed container directly in Kubernetes it is necessary to convert the checkpoint archive into an image that can be pushed to a registry.

```bash
yum install buildah -y

newcontainer=$(buildah from scratch)
buildah add $newcontainer /var/lib/kubelet/checkpoints/nginx.tar /
buildah config --annotation=io.kubernetes.cri-o.annotations.checkpoint.name=<container-name> $newcontainer
buildah commit $newcontainer checkpoint-image:latest
buildah rm $newcontainer

buildah push localhost/checkpoint-image:latest container-image-registry.example/user/checkpoint-image:latest
```

To restore this checkpoint image (container-image-registry.example/user/checkpoint-image:latest), the image needs to be listed in the specification for a Pod. Here's an example manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  namePrefix: example-
spec:
  containers:
  - name: <container-name>
    image: container-image-registry.example/user/checkpoint-image:latest
  nodeName: <destination-node>
```

