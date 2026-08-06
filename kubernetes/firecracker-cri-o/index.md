---
title: How to deploy CRI-O with Firecracker?
url: https://devopstales.github.io/kubernetes/firecracker-cri-o/
date: 2021-08-23
keywords: Kubernetes, kata-container, Firecracker, CRI-O, qvemu, kvm, AlmaLinux, devmapper, devicemapper
---


In this post I will show you how you can install and use kata-container with Firecracker engine in kubernetes.

<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

### What is Kata container engine

Kata Containers is an open source community working to build a secure container runtime with lightweight virtual machines that feel and perform like containers, but provide stronger workload isolation using hardware virtualization technology as a second layer of defense. (Source: [Kata Containers Website](https://katacontainers.io/) )

![Kata container engine](/img/include/katacontainers.webp)

### Why should you use Firecracker?

Firecracker is a way to run virtual machines, but its primary goal is to be used as a container runtime interface, making it use very few resources by design.

### Enable qvemu

I will use Vagrant and VirtualBox for running the AlmaLinux VM so first I need to enable then Nested virtualization on the VM:

```bash
VBoxManage modifyvm alma8 --nested-hw-virt on
```

After the Linux is booted test the virtualization flag in the VM:

```bash
egrep --color -i "svm|vmx" /proc/cpuinfo
```

If you find one of this flags everything is ok. Now we need to enable the kvm kernel module.

```bash
sudo modprobe kvm-intel
sudo modprobe vhost_vsock
```

### Disable selinux

```bash
setenforce 0
sed -i 's/=\(enforcing\|permissive\)/=disabled/g' /etc/sysconfig/selinux
sed -i 's/=\(enforcing\|permissive\)/=disabled/g' /etc/selinux/config
```

### Install and configure CRI-O

```bash
sudo dnf install epel-release nano wget -y

export VERSION=1.21
sudo curl -L -o /etc/yum.repos.d/devel_kubic_libcontainers_stable.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/CentOS_8/devel:kubic:libcontainers:stable.repo

sudo curl -L -o /etc/yum.repos.d/devel_kubic_libcontainers_stable_cri-o_${VERSION}.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:${VERSION}/CentOS_8/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo

yum install cri-o
```

Devmapper is the only storage driver supported by Firecracker. The stable pubic verion of `cri-o` dose not contained the support for devicemapper. Thanx to `Sascha Grunert` ther is a new version built with libdevmapper. He answered my question on slack immediately and created a patch for this bug.

```bash
nano /etc/containers/storage.conf
[storage]
driver = "devicemapper"
...
[storage.options.thinpool]
autoextend_percent = "20"
autoextend_threshold = "80"
basesize = "8G"
directlvm_device = "/dev/sdb"
directlvm_device_force = "True"
fs="xfs"

nano /etc/containers/registries.conf
registries = [
  "quay.io",
  "docker.io"
]
unqualified-search-registries = [
  "quay.io",
  "docker.io"
]

systemctl enable crio
systemctl restart crio
systemctl status crio

$ crio-status info
storage root: /var/lib/containers/storage
default GID mappings (format <container>:<host>:<size>):
  0:0:4294967295
default UID mappings (format <container>:<host>:<size>):
  0:0:4294967295
```

### Install tools

```bash
yum install git -y

sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/sbin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/sbin/kubens
```

### Install Kubernetes

Configure Kernel parameters for Kubernetes.

```bash
modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
kvm-intel
vhost_vsock
EOF

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

Disable swap for Kubernetes.

```bash
free -h
swapoff -a
swapoff -a
sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab
free -h
```

The I will add the kubernetes repo and Install the packages.

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

CRIP_VERSION=$(crio --version | awk '{print $3}')
yum install kubelet-$CRIP_VERSION kubeadm-$CRIP_VERSION kubectl-$CRIP_VERSION -y
```

You nee the same cgroup manager in cri-o and kubeadm. The default for kubeadm is cgroupfs and for cri-o the default is systemd. In this example I configured cri-o for cgroupfs.

```bash
nano /etc/crio/crio.conf
[crio.runtime]
conmon_cgroup = "pod"
cgroup_manager = "cgroupfs"

systemctl restart crio
```

If you want to use systemd:

```bash
echo "KUBELET_EXTRA_ARGS=--cgroup-driver=systemd" | tee /etc/sysconfig/kubelet
```

Start Kubernetes with containerd engine.

```bash
export IP=172.17.13.10

dnf install -y iproute-tc

systemctl enable kubelet.service

# for multi interface configuration
echo 'KUBELET_EXTRA_ARGS="--node-ip='$IP' --cgroup-driver=systemd"' > /etc/sysconfig/kubelet

kubeadm config images pull --cri-socket=unix:///var/run/crio/crio.sock --kubernetes-version=$CRIP_VERSION
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=$IP  --kubernetes-version=$CRIP_VERSION --cri-socket=unix:///var/run/crio/crio.sock
```

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get no
crictl ps

kubectl taint nodes $(hostname) node-role.kubernetes.io/master:NoSchedule-
```

### Initialize network

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl aplly -f kube-flannel.yml
```

OR

```bash
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
wget https://docs.projectcalico.org/manifests/custom-resources.yaml

nano custom-resources.yaml
...
      cidr: 10.244.0.0/16
...

kubectl apply -f custom-resources.yaml
```

### Install Kata container engine

If all the Nodes are ready deploy a Daemonsets to build Kata containers and firecracker wit `kata-deploy`:

```bash
$ kubectl get no

NAME    STATUS   ROLES                  AGE     VERSION
alma8   Ready    control-plane,master   2m31s   v1.22.1
```

```bash
# Installing the latest image

kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-rbac/base/kata-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-deploy/base/kata-deploy.yaml

# OR
# Installing the stable image

kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-rbac/base/kata-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-deploy/base/kata-deploy-stable.yaml
```

Verify the pod:

```bash
kubens kube-system

$ kubectl get DaemonSet
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kata-deploy   1         1         1       1            1           <none>                   3m43s
kube-proxy    1         1         1       1            1           kubernetes.io/os=linux   10m

$ kubectl get po kata-deploy-5zwmq
NAME                READY   STATUS    RESTARTS   AGE
kata-deploy-5zwmq   1/1     Running   0          4m24
```

```bash
kubectl logs kata-deploy-5zwmq
copying kata artifacts onto host
#!/bin/bash
KATA_CONF_FILE=/opt/kata/share/defaults/kata-containers/configuration-fc.toml /opt/kata/bin/containerd-shim-kata-v2 "$@"
#!/bin/bash
KATA_CONF_FILE=/opt/kata/share/defaults/kata-containers/configuration-qemu.toml /opt/kata/bin/containerd-shim-kata-v2 "$@"
#!/bin/bash
KATA_CONF_FILE=/opt/kata/share/defaults/kata-containers/configuration-clh.toml /opt/kata/bin/containerd-shim-kata-v2 "$@"
Add Kata Containers as a supported runtime for CRIO:

# Path to the Kata Containers runtime binary that uses the fc
[crio.runtime.runtimes.kata-fc]
	runtime_path = "/usr/local/bin/containerd-shim-kata-fc-v2"
	runtime_type = "vm"
	runtime_root = "/run/vc"
	privileged_without_host_devices = true

# Path to the Kata Containers runtime binary that uses the qemu
[crio.runtime.runtimes.kata-qemu]
	runtime_path = "/usr/local/bin/containerd-shim-kata-qemu-v2"
	runtime_type = "vm"
	runtime_root = "/run/vc"
	privileged_without_host_devices = true

# Path to the Kata Containers runtime binary that uses the clh
[crio.runtime.runtimes.kata-clh]
	runtime_path = "/usr/local/bin/containerd-shim-kata-clh-v2"
	runtime_type = "vm"
	runtime_root = "/run/vc"
	privileged_without_host_devices = true
node/alma8 labeled
```

```bash
$ ll /opt/kata/bin/
total 157532
-rwxr-xr-x. 1 root root  4045032 Jul 19 06:10 cloud-hypervisor
-rwxr-xr-x. 1 root root 42252997 Jul 19 06:12 containerd-shim-kata-v2
-rwxr-xr-x. 1 root root  3290472 Jul 19 06:14 firecracker
-rwxr-xr-x. 1 root root  2589888 Jul 19 06:14 jailer
-rwxr-xr-x. 1 root root    16686 Jul 19 06:12 kata-collect-data.sh
-rwxr-xr-x. 1 root root 37429099 Jul 19 06:12 kata-monitor
-rwxr-xr-x. 1 root root 54149384 Jul 19 06:12 kata-runtime
-rwxr-xr-x. 1 root root 17521656 Jul 19 06:18 qemu-system-x86_64
```

Restart containerd to enable the new config:

```bash
systemctl restart containerd
```

---

### Start Deployment

First I create a `RuntimeClass` for kata-fc then start a pod with this `RuntimeClass`.

```bash
kubens default

kubectl apply -f - <<EOF
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-fc
handler: kata-fc
EOF

cat<<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: untrusted
  name: www-kata-fc
spec:
  runtimeClassName: kata-fc
  containers:
  - image: nginx:1.18
    name: www
    ports:
    - containerPort: 80
EOF
```

```bash
kubectl apply -f - <<EOF
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-qemu
handler: kata-qemu
EOF

cat<<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: untrusted
  name: www-kata-qemu
spec:
  runtimeClassName: kata-qemu
  containers:
  - image: nginx:1.18
    name: www
    ports:
    - containerPort: 80
EOF
```

```bash
kubectl apply -f - <<EOF
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-clh
handler: kata-clh
EOF

cat<<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: untrusted
  name: www-kata-clh
spec:
  runtimeClassName: kata-clh
  containers:
  - image: nginx:1.18
    name: www
    ports:
    - containerPort: 80
EOF
```

```bash
$ kubectl get po
NAME            READY   STATUS    RESTARTS   AGE
www-kata-clh    1/1     Running   0          59s
www-kata-fc     1/1     Running   0          12s
www-kata-qemu   1/1     Running   0          69s
```


