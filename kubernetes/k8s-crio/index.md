---
title: Install K8S with CRI-O and kadalu
url: https://devopstales.github.io/kubernetes/k8s-crio/
date: 2020-09-04
keywords: kubernetes deployment, CRI-O, kadalu
---


In this post I will show you how to install cri-o container runtime and initialize a Kubernetes.

<!--more-->

{{< content "/filedir/kubernetes.html" >}}

### What is CRI-O?
The Kubernetes project has defined a number of standards. One of them is cri. The Container Runtime Interface. This interface defines how Kubernetes talks with a high-level container runtime. CRI-O is an implementation of the Kubernetes CRI to enable using OCI (Open Container Initiative) compatible runtimes. It is a lightweight alternative of Docker as the runtime for kubernetes. t allows Kubernetes to use any OCI-compliant runtime as the container runtime for running pods. Today it supports runc and Kata Containers as the container runtimes but any OCI-conformant runtime can be plugged in principle.

### Install CRI-O instad of Docker
```bash
VERSION=1.18
sudo curl -L -o /etc/yum.repos.d/devel_kubic_libcontainers_stable.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/CentOS_7/devel:kubic:libcontainers:stable.repo
sudo curl -L -o /etc/yum.repos.d/devel_kubic_libcontainers_stable_cri-o_${VERSION}.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:${VERSION}/CentOS_7/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo

yum install cri-o
```

### Configure
```bash
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

You nee the same cgroup manager in cri-o and kubeadm. The default for kubeadm is cgroupfs and for cri-o the default is systemd. In this example I configured cri-o for cgroupfs.

```bash
nano /etc/crio/crio.conf
[crio.runtime]
conmon_cgroup = "pod"
cgroup_manager = "cgroupfs"

nano /etc/containers/registries.conf
registries = [
  "quay.io",
  "docker.io"
]
```

If you want to use systemd:
```bash
echo "KUBELET_EXTRA_ARGS=--cgroup-driver=systemd" | tee /etc/sysconfig/kubelet
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

CRIP_VERSION=$(crio --version | awk '{print $3}')
yum install kubelet-$CRIP_VERSION kubeadm-$CRIP_VERSION kubectl-$CRIP_VERSION -y
```


```bash
IP=172.17.9.10

mkdir /var/lib/kubelet/

# --node-ip for multi interface configuration
# --cgroup-driver=systemd for systemd dryver

cat <<EOF > /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--node-ip='$IP' --cgroup-driver=systemd"
EOF
```

```bash
systemctl enable --now kubelet.service
systemctl enable --now cri-o
kubeadm config images pull --cri-socket=unix:///var/run/crio/crio.sock --kubernetes-version=$CRIP_VERSION
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=$IP  --kubernetes-version=$CRIP_VERSION --cri-socket=unix:///var/run/crio/crio.sock

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
kubectl aplly -f kube-flannel.yml
```

```bash
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
wget https://docs.projectcalico.org/manifests/custom-resources.yaml

nano ustom-resources.yaml
...
      cidr: 10.244.0.0/16
...

kubectl apply -f custom-resources.yaml
```

### Install tools
```bash
yum install git -y

sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/sbin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/sbin/kubens
```

### Deploy kadalu storage
```bash
sudo wipefs -a -t dos -f /dev/sdb
sudo mkfs.xfs /dev/sdb

yum install python3-pip -y
sudo pip3 install kubectl-kadalu

echo "export PATH=$PATH:/usr/local/bin/" >> /etc/profile
source /etc/profile

kubectl kadalu install

# k8s.mydomain.intra is the nod name in Kubernetes
# /dev/sdb is the disk

kubectl kadalu storage-add storage-pool-1 \
    --device k8s.mydomain.intra:/dev/sdb

# to delete object if you misconfigured kadalu
kubectl delete kadalustorages.kadalu-operator.storage storage-pool-1


kubectl get pods -n kadalu

kubectl patch storageclass kadalu.replica1 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

```bash
nano test-pvc.yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv1
spec:
  storageClassName: kadalu.replica1
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

