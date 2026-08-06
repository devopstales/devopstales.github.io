---
title: Install K3S with CRI-O and kadalu
url: https://devopstales.github.io/kubernetes/k3s-crio/
date: 2020-09-10
keywords: kubernetes deployment, CRI-O, Rancher, K3S, kadalu
---


In this post I will show you how to install cri-o container runtime and initialize a Kubernetes.

<!--more-->

{{< content "/filedir/k3s.html" >}}

### Install CRI-O instad of Docker
```
VERSION=1.18
sudo curl -L -o /etc/yum.repos.d/devel_kubic_libcontainers_stable.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/CentOS_7/devel:kubic:libcontainers:stable.repo
sudo curl -L -o /etc/yum.repos.d/devel_kubic_libcontainers_stable_cri-o_${VERSION}.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:${VERSION}/CentOS_7/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo

yum install cri-o
```

### Configure
```
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

```
free -h
swapoff -a
swapoff -a
sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab
free -h
```

You nee the same cgroup manager in cri-o and kubeadm. The default for kubeadm is cgroupfs and for cri-o the default is systemd. In this example I configured cri-o for cgroupfs.

```
nano /etc/crio/crio.conf
[crio.runtime]
conmon_cgroup = "pod"
cgroup_manager = "cgroupfs"
...
registries = [
  "quay.io",
  "docker.io"
]
```

Disable ipv6 and configure cri-o CNI confg for flanel's network:
```
echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.conf
sysctl -p

sed -i "s|::1|#::1|" /etc/hosts

nano /etc/cni/net.d/100-crio-bridge.conf
{
...
"ipam": {
    "type": "host-local",
    "routes": [
        { "dst": "0.0.0.0/0" }
    ],

    "ranges": [
        [{ "subnet": "10.244.0.0/16" }]
    ]
}
}
```

```
systemctl enable --now cri-o

echo "export PATH=$PATH:/usr/local/bin/" >> /etc/profile
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> /etc/profile
source /etc/profile

yum install -y container-selinux selinux-policy-base
rpm -i https://rpm.rancher.io/k3s-selinux-0.1.1-rc1.el7.noarch.rpm
```

```
export K3S_KUBECONFIG_MODE="644"
export INSTALL_K3S_EXEC=" --container-runtime-endpoint /var/run/crio/crio.sock --no-deploy servicelb --no-deploy traefik"

curl -sfL https://get.k3s.io | sh -
```

```
systemctl status k3s

crictl info
crictl ps
kubectl get node -o wide
kubectl get pods -A -o wide
```

### Install tools
```
yum install git -y

sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/sbin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/sbin/kubens
COMPDIR=$(pkg-config --variable=completionsdir bash-completion)
ln -sf /opt/kubectx/completion/kubens.bash $COMPDIR/kubens
ln -sf /opt/kubectx/completion/kubectx.bash $COMPDIR/kubectx
```

### Deploy kadalu storage
```
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

```
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

