---
title: kubernetes 1.24: Install cri-dockerd for docker
url: https://devopstales.github.io/kubernetes/migrate-kubernetes-to-dockershim/
date: 2022-05-05
keywords: kubernetes deployment, kubernetes vs docker, docker, dockershim, migrate, switch, cri-dockerd
---


With the new Kubernetes 1.24 and deprecation of dockershim, in this post I will show you how you can migrate your kubernetes cluster to use cri-dockerd instad of dockershim.

<!--more-->

### What is dockershim?
Dockershim was a componeni in the kubernetes engine that translated between the docker api and the Kubernetes. With the new Kubernetes 1.24 Kubernetes removed this component from it's codebase. Now only the CRI (Container Runetime Interface) compatible container engines can be used. Mirantys and Docker decided to move dockershim to another repository. As part of this project, initial steps were taken to wrap Dockershim in something that can speak CRI.

### So, what is cri-dockerd?
Well, for now, it’s a thin wrapper around Dockershim itself which allows Dockershim to be started as a separate daemon. That daemon presents the expected API for kubelet’s That daemon presents the expected API for kubelet’s.

### How to migrate

You have to be careful if you are on a single master node configuration. The cluster will be unavailable under the upgrade.

```bash
kubectl get nodes -o wide
NAME    STATUS   ROLES                  AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION                             CONTAINER-RUNTIME
k8s01   Ready    control-plane,master   78m     v1.20.4   10.65.79.164   <none>        CentOS Linux 8       4.18.0-240.15.1.el8_3.centos.plus.x86_64   docker://20.10.5
k8s02   Ready    control-plane,master   64m     v1.20.4   10.65.79.131   <none>        CentOS Linux 8       4.18.0-240.15.1.el8_3.centos.plus.x86_64   docker://20.10.5
k8s03   Ready    control-plane,master   4m16s   v1.20.4   10.65.79.244   <none>        CentOS Linux 8       4.18.0-240.15.1.el8_3.centos.plus.x86_64   docker://20.10.5
```

First we will cordon and drain the node:

```bash
kubectl cordon k8s01
kubectl drain k8s01 --ignore-daemonsets
```

```bash
kubectl get nodes
NAME    STATUS                      ROLES                  AGE    VERSION
k8s01   Ready,SchedulingDisabled    control-plane,master   83m    v1.20.4
k8s02   Ready                       control-plane,master   69m    v1.20.4
k8s03   Ready                       control-plane,master   9m30s  v1.20.4
```

Stop the kubelet sevice:

```bash
sudo systemctl stop kubelet
sudo systemctl status kubelet
```

### Install and configure dockershim:

```bash
cd /tmp
VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest|grep tag_name | cut -d '"' -f 4)
echo $VER

wget https://github.com/Mirantis/cri-dockerd/releases/download/${VER}/cri-dockerd-${VER}-linux-amd64.tar.gz
tar xvf cri-dockerd-${VER}-linux-amd64.tar.gz

sudo mv cri-dockerd /usr/local/bin/

cri-dockerd --version
cri-dockerd 0.2.0 (HEAD)

wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
```

### Configure the kubelet to use cri-dockerd

test the socket connection:

```bash
sudo kubeadm config images pull --cri-socket /run/cri-dockerd.sock 
[config/images] Pulled k8s.gcr.io/kube-apiserver:v1.23.5
[config/images] Pulled k8s.gcr.io/kube-controller-manager:v1.23.5
[config/images] Pulled k8s.gcr.io/kube-scheduler:v1.23.5
[config/images] Pulled k8s.gcr.io/kube-proxy:v1.23.5
[config/images] Pulled k8s.gcr.io/pause:3.6
[config/images] Pulled k8s.gcr.io/etcd:3.5.1-0
[config/images] Pulled k8s.gcr.io/coredns/coredns:v1.8.6
```

```bash
nano /etc/sysconfig/kubelet
# add the following flags to KUBELET_KUBEADM_ARGS variable
KUBELET_KUBEADM_ARGS="... --container-runtime=remote --container-runtime-endpoint=/run/cri-dockerd.sock"
```

Start kubelet:

```bash
sudo systemctl start kubelet
```

Check if the new runtime on the node:

```bash
kubectl describe node k8s01
```

Uncordon the node to mark it schedulable agen:

```bash
kubectl uncordon k8s01
```

Once you changed the runetime on all the nodes you are done.

