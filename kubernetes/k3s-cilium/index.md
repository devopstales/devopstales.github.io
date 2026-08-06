---
title: Install K3S with k3sup and Cilium
url: https://devopstales.github.io/kubernetes/k3s-cilium/
date: 2021-04-17
keywords: kubernetes deployment, Rancher, K3S, k3sup, Cilium
---


In this post I will show you how to install K3S with k3sup and use Cilium as networking.

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

### Install cilium for networking

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
tmux-cssh -u vagrant 172.17.8.101 172.17.8.102 172.17.8.103


sudo mount bpffs -t bpf /sys/fs/bpf
sudo bash -c 'cat <<EOF >> /etc/fstab
none /sys/fs/bpf bpf rw,relatime 0 0
EOF'

exit
```

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

kubectl create -n kube-system secret generic cilium-ipsec-keys \
    --from-literal=keys="3 rfc4106(gcm(aes)) $(echo $(dd if=/dev/urandom count=20 bs=1 2> /dev/null| xxd -p -c 64)) 128"


kubectl -n kube-system get secrets cilium-ipsec-keys
```

```bash
nano values.yaml
---
kubeProxyReplacement: "strict"

k8sServiceHost: 10.0.2.15
k8sServicePort: 6443

global:
  encryption:
    enabled: true
    nodeEncryption: true

hubble:
  metrics:
    #serviceMonitor:
    #  enabled: true
    enabled:
    - dns:query;ignoreAAAA
    - drop
    - tcp
    - flow
    - icmp
    - http

  ui:
    enabled: true
    replicas: 1
    ingress:
      enabled: true
      hosts:
        - hubble.k3s.intra
      annotations:
        cert-manager.io/cluster-issuer: ca-issuer
      tls:
      - secretName: ingress-hubble-ui
        hosts:
        - hubble.k3s.intra

  relay:
    enabled: true


operator:
  replicas: 1

ipam:
  mode: "cluster-pool"
  operator:
    clusterPoolIPv4PodCIDR: "10.43.0.0/16"
    clusterPoolIPv4MaskSize: 24
    clusterPoolIPv6PodCIDR: "fd00::/104"
    clusterPoolIPv6MaskSize: 120

prometheus:
  enabled: true
  # Default port value (9090) needs to be changed since the RHEL cockpit also listens on this port.
  port: 19090
```

```bash
helm upgrade --install cilium cilium/cilium   --namespace kube-system -f values.yaml
```

```bash
k get po -A   
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   cilium-operator-67895d78b7-vkgcs          1/1     Running   0          89s
kube-system   cilium-zppdd                              1/1     Running   0          89s
kube-system   coredns-854c77959c-b4gzq                  1/1     Running   0          40s
kube-system   local-path-provisioner-5ff76fc89d-9xjgz   1/1     Running   0          40s
kube-system   metrics-server-86cbb8457f-t4d6l           1/1     Running   0          40s
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

### Enable Hubble for Cluster-Wide Visibility

I configured an ingress with https in cilium helm chart but you can use port-forward instead of that.

```bash
kubectl port-forward -n kube-system svc/hubble-ui \
--address 0.0.0.0 --address :: 12000:80
```

And then open http://localhost:12000/ to access the UI.

