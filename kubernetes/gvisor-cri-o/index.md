---
title: How to deploy CRI-O with gVisor?
url: https://devopstales.github.io/kubernetes/gvisor-cri-o/
date: 2021-08-23
keywords: Kubernetes, gvisor, CRI-O, qvemu, AlmaLinux
---


In this post I will show you how you can install and use gvisor engine in kubernetes.

<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

### What is gvisor

gVisor is an application kernel, written in Go, that implements a substantial portion of the Linux system call interface. It provides an additional layer of isolation between running applications and the host operating system.

gVisor includes an Open Container Initiative (OCI) runtime called `runsc` that makes it easy to work with existing container tooling. The `runsc` runtime integrates with Docker, CRI-O and Kubernetes, making it simple to run sandboxed containers.

![gvisor](/img/include/gvisor2.webp)
![gvisor](/img/include/gvisor.webp)

### Install gvisor

```bash
sudo dnf install epel-release nano wget -y

nano gvisor.sh
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
```

```bash
bash gvisor.sh
...
runsc: OK
containerd-shim-runsc-v1: OK
```

### Install and configure CRI-O

```bash
export VERSION=1.21
sudo curl -L -o /etc/yum.repos.d/devel_kubic_libcontainers_stable.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/CentOS_8/devel:kubic:libcontainers:stable.repo

sudo curl -L -o /etc/yum.repos.d/devel_kubic_libcontainers_stable_cri-o_${VERSION}.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:${VERSION}/CentOS_8/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo

yum install cri-o
```

`runsc` implements cgroups using `cgroupfs` so I will use `cgroupfs` in `CRI-O` and Kubernets config.

```bash
nano /etc/crio/crio.conf
[crio.runtime]
conmon_cgroup = "pod"
cgroup_manager = "cgroupfs"
selinux = false

nano /etc/containers/registries.conf
registries = [
  "quay.io",
  "docker.io"
]
unqualified-search-registries = [
  "quay.io",
  "docker.io"
]
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
cat <<EOF | sudo tee /etc/modules-load.d/CRI-O.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
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

Start Kubernetes with CRI-O engine.

```bash
export IP=172.17.13.10

dnf install -y iproute-tc

systemctl enable kubelet.service

# for multi interface configuration
echo 'KUBELET_EXTRA_ARGS="--node-ip='$IP' --cgroup-driver=cgroupfs"' > /etc/sysconfig/kubelet

kubeadm config images pull --cri-socket=unix:///var/run/crio/crio.sock --kubernetes-version=$CRIP_VERSION
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=$IP --kubernetes-version=$CRIP_VERSION --cri-socket=unix:///var/run/crio/crio.sock
```

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get no
crictl ps

kubectl taint nodes $(hostname) node-role.kubernetes.io/master:NoSchedule-
```

### Inincialize network

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

### Start Deployment

First I create a `RuntimeClass` for gvisor then start a pod with this `RuntimeClass`.

```bash
cat<<EOF | kubectl apply -f -
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
EOF

cat<<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: untrusted
  name: www-gvisor2
spec:
  runtimeClassName: gvisor
  containers:
  - image: nginx:1.18
    name: www
    ports:
    - containerPort: 80
EOF
```

```bash
$ kubectl get po
NAME        READY   STATUS    RESTARTS   AGE
www-gvisor  1/1     Running   0          2m47s


$ kubectl describe po www-gvisor
...
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m42s  default-scheduler  Successfully assigned default/www-kata to alma8
  Normal  Pulled     2m13s  kubelet            Container image "nginx:1.18" already present on machine
  Normal  Created    2m13s  kubelet            Created container www
  Normal  Started    2m11s  kubelet            Started container www
```

