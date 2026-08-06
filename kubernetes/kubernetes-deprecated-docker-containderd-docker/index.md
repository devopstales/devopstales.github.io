---
title: Kubernetes deprecated Docker? Containderd is the new Docker!!
url: https://devopstales.github.io/kubernetes/kubernetes-deprecated-docker-containderd-docker/
date: 2020-12-04
keywords: kubernetes deployment, Containerd, Docker is deprecated in Kubernetes now?, Containderd is the new Docker!!
---


Docker is now deprecated in Kubernetes in the next 1.20 version, but thet dose no mean yo can not run containers wit docker.

<!--more-->

"Given the impact of this change, we are using an extended deprecation timeline. It will not be removed before Kubernetes 1.22, meaning the earliest release without dockershim would be 1.23 in late 2021." (Source: [Kubernetes](https://kubernetes.io/blog/2020/12/02/dockershim-faq/) )

### But why is Docker deprecated?
In the beginning Kubernetes supported only Docker as a container runtime but to use other runtime's Kubernetes, Docker, Google, CoreOS, and other vendors created the [Open Container Initiative (OCI)](https://opencontainers.org/). The OCI currently contains two specifications: the Runtime Specification (runtime-spec)  and the Image Specification (image-spec).
For the Runtime Specification they created the CRI (Container runtime Interface) as a standerd interface fro kubernetes to communicate with container runtimes. Before 1.20 Kubernetes used the old dockershim for docker engine not the standerd CRI interface.

To explain the next reason, we have to see the Docker architecture a bit. Here's the diagram.

![Example image](/img/include/docker_engine.webp)

Kubernetes needs the tings inside of the red area. Docker has many other features like Docker Network and Volume that Kubernetes not uses.

### Can I use Docker??
Mirantis and Docker have agreed to partner to maintain the shim code standalone outside Kubernetes, as a conformant CRI interface for Docker Engine. ... This means that you can continue to build Kubernetes based on Docker Engine as before, just switching from the built in dockershim to the external one. Docker and Mirantis will work together on making sure it continues to work as well as before and that it passes all the conformance tests and works just like the built in version did. Docker will continue to ship this shim in Docker Desktop as this gives a great developer experience, and Mirantis will be using this in Mirantis Kubernetes Engine. (Source: [Docker Blog](https://www.docker.com/blog/what-developers-need-to-know-about-docker-docker-engine-and-kubernetes-v1-20/) )

### What can I use instad of Docker
You can use containerd or CRI-O instad of docker. In [a previous post]() I showed how you can install Kubernetes with CRI-O, so now I will show you how you can use containerd instad of Docker. If you just want to migrate from Docker, this is the best option as containerd is actually used inside of Docker to do all the "runtime" jobs as you can see in the diagram above.

### Install and configure Containerd prerequisites:
```yaml
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

```bash
### Install required packages
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

## Add docker repository
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

## Install containerd
sudo yum update -y && sudo yum install -y containerd.io

## Configure containerd
sudo mkdir -p /etc/containerd
sudo containerd config default > /etc/containerd/config.toml
```

To use the `systemd` cgroup driver in `/etc/containerd/config.toml` with `runc`, set
```bash
nano /etc/containerd/config.toml
systemd_cgroup = true

# Restart containerd
sudo systemctl restart containerd
```

```bash
free -h
swapoff -a
swapoff -a
sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab
free -h
```

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF


yum install epel-release -y
yum install -y kubeadm kubelet kubectl
```

```bash
echo "runtime-endpoint: unix:///run/containerd/containerd.sock" > /etc/crictl.yaml
crictl ps
```

```bash
systemctl enable kubelet.service
kubeadm config images pull
crictl images
kubeadm init --pod-network-cidr=10.244.0.0/16
```

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl get node
```

