---
title: Install K3S with k3sup and Calico
url: https://devopstales.github.io/kubernetes/k3sup-calico/
date: 2021-04-18
keywords: kubernetes deployment, Rancher, K3S, k3sup, Calico
---


In this post I will show you how to install K3S with k3sup and use Calico as networking.

<!--more-->

{{< content "/filedir/k3s.html" >}}

### Installing k3sup

```bash
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/
k3sup --help
```

```bash
ssh-copy-id vagrant@172.17.8.101
ssh-copy-id vagrant@172.17.8.102
ssh-copy-id vagrant@172.17.8.103
```

### Bootstrap the first k3s node

```bash
k3sup install \
  --ip=172.17.8.101 \
  --user=vagrant \
  --sudo \
  --cluster \
  --k3s-channel=stable \
  --k3s-extra-args "--flannel-backend=none --cluster-cidr=10.10.0.0/16 --disable-network-policy --no-deploy=traefik --no-deploy=servicelb --node-ip=172.17.8.101" \
  --merge \
  --local-path $HOME/.kube/config \
  --context=k3s-ha
```

### Install calico for networking

```bash
kubectx k3s-ha

kubectl get no
NAME        STATUS     ROLES                       AGE   VERSION
k3s-node1   NotReady   control-plane,etcd,master   15m   v1.20.5+k3s1

kubectl get po -A -o wide
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   coredns-854c77959c-zbgkt                  0/1     Pending   0          16m
kube-system   local-path-provisioner-5ff76fc89d-btmx6   0/1     Pending   0          16m
kube-system   metrics-server-86cbb8457f-n99rp           0/1     Pending   0          16m
```

```bash
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```

```bash
kubectl get po -A   
NAMESPACE         NAME                                       READY   STATUS    RESTARTS   AGE
calico-system     calico-kube-controllers-77c7dbc6d6-srss8   1/1     Running   0          90s
calico-system     calico-node-zd7rg                          1/1     Running   0          90s
calico-system     calico-typha-7b4c95fcd4-lw4wx              1/1     Running   0          90s
kube-system       coredns-854c77959c-zbgkt                   1/1     Running   0          27m
kube-system       local-path-provisioner-5ff76fc89d-btmx6    1/1     Running   0          27m
kube-system       metrics-server-86cbb8457f-n99rp            1/1     Running   0          27m
tigera-operator   tigera-operator-675ccbb69c-fv894           1/1     Running   0          10m
```

### Bootstrap the other k3s nodes

```bash
k3sup join \
  --ip 172.17.8.102 \
  --user vagrant \
  --sudo \
  --k3s-channel stable \
  --server \
  --server-ip 172.17.8.101 \
  --server-user vagrant \
  --sudo \
  --k3s-extra-args "--flannel-backend=none --cluster-cidr=10.10.0.0/16 --disable-network-policy --no-deploy=traefik --no-deploy=servicelb --node-ip=172.17.8.102"
  
k3sup join \
  --ip 172.17.8.103 \
  --user vagrant \
  --sudo \
  --k3s-channel stable \
  --server \
  --server-ip 172.17.8.101 \
  --server-user vagrant \
  --sudo \
  --k3s-extra-args "--flannel-backend=none --cluster-cidr=10.10.0.0/16 --disable-network-policy --no-deploy=traefik --no-deploy=servicelb --node-ip=172.17.8.103"
```

