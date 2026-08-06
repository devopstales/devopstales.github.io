---
title: Kubernetes Hardening Guide with CISA 1.6 Benchmark
url: https://devopstales.github.io/kubernetes/k8s-cisa-install/
date: 2021-10-15
keywords: kubernetes deployment, seccomp, docker, containerd, CRI-O, Kubernetes, CISA, NSA, best practices, Kubernetes Hardening Guide
---

On August 3rd, 2021 the National Security Agency (NSA) and the Cybersecurity and Infrastructure Security Agency (CISA) released, [Kubernetes Hardening Guidance](https://media.defense.gov/2021/Aug/03/2002820425/-1/-1/1/CTR_KUBERNETES%20HARDENING%20GUIDANCE.PDF), a cybersecurity technical report detailing the complexities of securely managing Kubernetes. This blog post will show you how you can harden your Kubernetes cluster based on CISA best practices. 

<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

### Disable swap

```bash
free -h
swapoff -a
swapoff -a
sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab
free -h
```

### Project Longhorn Prerequisites

```bash
yum install -y iscsi-initiator-utils 
modprobe iscsi_tcp
echo "iscsi_tcp" >/etc/modules-load.d/iscsi-tcp.conf
systemctl enable iscsid
systemctl start iscsid 
```

### Install and configure containerd

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2 git nano wget iproute-tc vim-common
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum install -y containerd.io

## Configure containerd
sudo mkdir -p /etc/containerd
sudo containerd config default > /etc/containerd/config.toml

nano /etc/containerd/config.toml
...
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true

systemctl enable --now containerd
systemctl status containerd

echo "runtime-endpoint: unix:///run/containerd/containerd.sock" > /etc/crictl.yaml

cd /tmp
wget https://github.com/containerd/nerdctl/releases/download/v0.12.0/nerdctl-0.12.0-linux-amd64.tar.gz

tar -xzf nerdctl-0.12.0-linux-amd64.tar.gz
mv nerdctl /usr/local/bin
nerdctl ps
```

### kubaedm preConfiguuration

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
#
# protectKernelDefaults
#
kernel.keys.root_maxbytes           = 25000000
kernel.keys.root_maxkeys            = 1000000
kernel.panic                        = 10
kernel.panic_on_oops                = 1
vm.overcommit_memory                = 1
vm.panic_on_oom                     = 0
EOF

sysctl --system
```

```bash
cd /opt
git clone https://github.com/devopstales/k8s_sec_lab

mkdir /etc/kubernetes/

head -c 32 /dev/urandom | base64
nano /opt/k8s_sec_lab/k8s-manifest/002-etcd-encription.yaml

cp /opt/k8s_sec_lab/k8s-manifest/001-audit-policy.yaml /etc/kubernetes/audit-policy.yaml
cp /opt/k8s_sec_lab/k8s-manifest/002-etcd-encription.yaml /etc/kubernetes/etcd-encription.yaml
```

### Install kubeadm

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


yum install epel-release

yum install -y kubeadm kubelet kubectl

echo 'KUBELET_KUBEADM_ARGS="--node-ip=172.17.13.10 --container-runtime=remote --container-runtime-endpoint=/run/containerd/containerd.sock"' > /etc/sysconfig/kubelet

kubeadm config images pull --config 010-kubeadm-conf-1-22-2.yaml

systemctl enable kubelet.service

nano 010-kubeadm-conf.yaml
# add the ips, short hostnames, fqdns of the nodes and the ip of the loadbalancer as certSANs to the apiServer config.

kubeadm init --skip-phases=addon/kube-proxy --config 010-kubeadm-conf-1-22-2.yaml

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# In my kubeadm config I forced the usage of PSP.
# At the beginning there is no psp deployed, so non of the pods can start.
# Thi is tru for the kube-apiserver too.

kubectl apply -f 011-psp.yaml 
```

```bash
kubectl get csr --all-namespaces

kubectl get csr -oname | xargs kubectl certificate approve

kubectl apply -f 012-k8s-clusterrole.yaml
```

### cilium

```bash
mount | grep /sys/fs/bpf
```

```bash
yum install -y https://harbottle.gitlab.io/harbottle-main/7/x86_64/harbottle-main-release.rpm
yum install -y kubectx helm

helm repo add cilium https://helm.cilium.io/

kubectl taint nodes --all node-role.kubernetes.io/master-


helm upgrade --install cilium cilium/cilium \
  --namespace kube-system \
  -f 031-cilium-helm-values.yaml

kubectl get pods -A

kubectl apply -f 013-k8s-cert-approver.yaml
```

### harden kubernetes

There is an opensource tool theat tests CISA's best best practices on your clsuter. We vill use this to test the resoults. 

```bash
# kube-bench
# https://github.com/aquasecurity/kube-bench/releases/
yum install -y https://github.com/aquasecurity/kube-bench/releases/download/v0.6.5/kube-bench_0.6.5_linux_amd64.rpm
```

```bash
useradd -r -c "etcd user" -s /sbin/nologin -M etcd
chown etcd:etcd /var/lib/etcd
chmod 700 /var/lib/etcd

# kube-bench
kube-bench
kube-bench | grep "\[FAIL\]"
```

There is no FAIL jusk WARNING. Jeee.

### join nodes

Firs we need to get the join command from the master:

```bash
# on master1
kubeadm token create --print-join-command
kubeadm join 172.17.9.10:6443 --token c2t0rj.cofbfnwwrb387890 \
 --discovery-token-ca-cert-hash sha256:a52f4c16a6ce9ef72e3d6172611d17d9752dfb1c3870cf7c8ad4ce3bcb97547e
```

If the next node is a worke we can just use the command what we get. If a next node is a master we need to generate a certificate-key. You need a separate certificate-key for every new master.

```bash
# on master1
## generate cert key
kubeadm certs certificate-key
29ab8a6013od73s8d3g4ba3a3b24679693e98acd796356eeb47df098c47f2773

## store cert key in secret
kubeadm init phase upload-certs --upload-certs --certificate-key=29ab8a6013od73s8d3g4ba3a3b24679693e98acd796356eeb47df098c47f2773
```

```bash
# on master2
kubeadm join 172.17.9.10:6443 --token c2t0rj.cofbfnwwrb387890 \
--discovery-token-ca-cert-hash sha256:a52f4c16a6ce9ef72e3d6172611d17d9752dfb1c3870cf7c8ad4ce3bcb97547e \
--control-plane --certificate-key 29ab8a6013od73s8d3g4ba3a3b24679693e98acd796356eeb47df098c47f2773
```

```bash
# on master3
kubeadm join 172.17.9.10:6443 --token c2t0rj.cofbfnwwrb387890 \
--discovery-token-ca-cert-hash sha256:a52f4c16a6ce9ef72e3d6172611d17d9752dfb1c3870cf7c8ad4ce3bcb97547e \
--control-plane --certificate-key 29ab8a6013od73s8d3g4ba3a3b24679693e98acd796356eeb47df098c47f2773
```

In the end withevery new node we need to approve the certificate requests for the node.

```bash
kubectl get csr -oname | xargs kubectl certificate approve

useradd -r -c "etcd user" -s /sbin/nologin -M etcd
chown etcd:etcd /var/lib/etcd
chmod 700 /var/lib/etcd
```

