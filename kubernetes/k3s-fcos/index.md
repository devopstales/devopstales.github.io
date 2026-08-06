---
title: Install K3S on Fedora CoreOS
url: https://devopstales.github.io/kubernetes/k3s-fcos/
date: 2020-09-05
keywords: Fedora CoreOS, FCOS, Rancher, K3S
---


In this post I will show you how you can Install K3S on Fedora CoreOS(FCOS) in virtualization environment.

<!--more-->

{{< content "/filedir/k3s.html" >}}

### What is K3S
K3S is a lightweight certified kubernetes distribution. Designed to be a single binary of less than 40MB. It didn't use etcd instad stora it's data in sqlite.

### Install requirements
```
sudo -i
rpm-ostree install https://rpm.rancher.io/k3s-selinux-0.1.1-rc1.el7.noarch.rpm
systemctl reboot
```

### Install K3S
```
sudo -i
export K3S_KUBECONFIG_MODE="644"
export INSTALL_K3S_EXEC=" --no-deploy servicelb --no-deploy traefik"

curl -sfL https://get.k3s.io | sh -

systemctl status k3s
kubectl get nodes -o wide
kubectl get pods -A -o wide
```

### Join other nodes
First we need the join token:
```
sudo cat /var/lib/rancher/k3s/server/node-token
K1042e2f8e353b9409472c1e0cca8457abe184dc7be3f0805109e92c50c193ceb42::node:c83acbf89a7de7026d6f6928dc270028
```

The join the worker nodes:
```
export K3S_KUBECONFIG_MODE="644"
export K3S_URL="https://k3s-master:6443"
export K3S_TOKEN="K1042e2f8e353b9409472c1e0cca8457abe184dc7be3f0805109e92c50c193ceb42::node:c83acbf89a7de7026d6f6928dc270028"

curl -sfL https://get.k3s.io | sh -
```

