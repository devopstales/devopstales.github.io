---
title: Install kubernetes with kubeadm
url: https://devopstales.github.io/kubernetes/k8s-install/
date: 2019-07-12
keywords: kubernetes deployment, kubernetes vs docker
---


Kubeadm is a tool that helps you bootstrap a simple Kubernetes cluster and simplifies the deployment process.

<!--more-->

{{< content "/filedir/kubernetes.html" >}}

```
192.168.1.41  kubernetes01 # master node
192.168.1.42  kubernetes02 # frontend node
192.168.1.43  kubernetes03 # worker node
192.168.1.44  kubernetes04 # worker node
192.168.1.45  kubernetes05 # worker node

# hardware requirement
4 CPU
16G RAM
```

### Install Docker
```
apt-get update
apt-get install apt-transport-https ca-certificates curl gnupg2 software-properties-common
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io
systemct start docker
systemct enable docker
```

### Configuuration
```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
EOF

echo '1' > /proc/sys/net/ipv4/ip_forward
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```

### Disable swap
```
free -h
swapoff -a
swapoff -a
sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab
free -h
```

### Install kubeadm
```
apt-get install ebtables ethtool apt-transport-https

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update && apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubect
systemctl enable kubelet && systemctl start kubelet
kubeadm config images pul
```

### Init master
```
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.1.41


mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```

### Join workers to cluster
```
kubeadm join 192.168.1.41:6443 --token XXXXXXXX \
    --discovery-token-ca-cert-hash sha256:XXXXXXXX
```

