---
title: Install K3S with k3sup and kube-vip
url: https://devopstales.github.io/kubernetes/k3s-etcd-kube-vip/
date: 2021-04-16
keywords: kubernetes deployment, Rancher, K3S, k3sup, kube-vip, kube vip, kubevip
---


In this post I will show you how to install K3S with k3sup. I will use kube-vip for  High-Availability and load-balancing.

<!--more-->

{{< content "/filedir/k3s.html" >}}

### The infrastructure

```bash
k3s-node1:
ip: 172.17.8.101
etcd
kube-vip

k3s-node2:
ip: 172.17.8.102
etcd
kube-vip

k3s-node3:
ip: 172.17.8.103
etcd
kube-vip
```

### What is k3sup?

K3S dose not give you an rpm or deb installer option just a binary. To install you need to create the systemd service and configure it. For a big cluster 3 or 5 node it could be a pain. k3sup automates this tasks trout ssh. You need a passwordless ssh connection for all the nodes and the k3sup binary on your computer.

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
  --tls-san=172.17.8.100 \
  --cluster \
  --k3s-channel=stable \
  --k3s-extra-args "--no-deploy=traefik --no-deploy=servicelb --flannel-iface=enp0s8 --node-ip=172.17.8.101" \
  --merge \
  --local-path $HOME/.kube/config \
  --context=k3s-ha
```

I used the `--tls-san` option to add the LoadBalancer's virtual ip to the cert, and a few extra option. I disabled the  traefik and the servicelb service because I will use nginx ingress controller and kube-vip as loadbalancer.  In my environment I used Vangrant to spin up the nodes.Vagrant creats multiple interfaces for the vm so I need to configure which of these will be used for the cluster: `--flannel-iface=enp0s8 --node-ip=172.17.8.101` Thanks to the `--cluster` k3sup will start an embedded etcd cluster in a container.

### Install kube-vip for HA

```bash
kubectx k3s-ha

kubectl get nodes

kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
```

ssh to the first host and generate the daemonset to run kube-vip:

```bash
ssh vagrant@172.17.8.101
sudo su -

ctr image pull docker.io/plndr/kube-vip:0.3.2
alias kube-vip="ctr run --rm --net-host docker.io/plndr/kube-vip:0.3.2 vip /kube-vip"

kube-vip manifest daemonset \
    --arp \
    --interface enp0s8 \
    --address 172.17.8.100 \
    --controlplane \
    --leaderElection \
    --taint \
    --inCluster | tee /var/lib/rancher/k3s/server/manifests/kube-vip.yaml

exit
```

Test vip:

```bash
ping 172.17.8.100
PING 172.17.8.100 (172.17.8.100) 56(84) bytes of data.
64 bytes from 172.17.8.100: icmp_seq=1 ttl=64 time=1.06 ms
64 bytes from 172.17.8.100: icmp_seq=2 ttl=64 time=0.582 ms
64 bytes from 172.17.8.100: icmp_seq=3 ttl=64 time=0.773 ms
```

### Bootstrap the other k3s nodes

```bash
k3sup join \
  --ip 172.17.8.102 \
  --user vagrant \
  --sudo \
  --k3s-channel stable \
  --server \
  --server-ip 172.17.8.100 \
  --server-user vagrant \
  --sudo \
  --k3s-extra-args "--no-deploy=traefik --no-deploy=servicelb --flannel-iface=enp0s8 --node-ip=172.17.8.102"
  
k3sup join \
  --ip 172.17.8.103 \
  --user vagrant \
  --sudo \
  --k3s-channel stable \
  --server \
  --server-ip 172.17.8.100 \
  --server-user vagrant \
  --sudo \
  --k3s-extra-args "--no-deploy=traefik --no-deploy=servicelb --flannel-iface=enp0s8 --node-ip=172.17.8.103"
```

### What is kube-vip

Kubernetes does not offer an implementation of network load-balancers (Services of type LoadBalancer) for bare metal clusters. The implementations of Network LB that Kubernetes does ship with are all glue code that calls out to various IaaS platforms (GCP, AWS, Azure…). If you’re not running on a supported IaaS platform (GCP, AWS, Azure…), LoadBalancers will remain in the "pending" state indefinitely when created. So I will use kube-vip to solve this problem.

MetalLB is also a popular tool for on-premises Kubernetes networking, however its primary use-case is for advertising service LoadBalancers instead of advertising a stable IP for the control-plane. kube-vip handles both use-cases, and is under active development by its author, Dan.

### Install kube-vip as network LoadBalancer

```bash
kubectl apply -f https://kube-vip.io/manifests/controller.yaml

kubectl create configmap --namespace kube-system plndr --from-literal cidr-global=172.17.8.200/29

wget https://kube-vip.io/manifests/kube-vip.yaml
nano kube-vip.yaml
...
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: vip-role
rules:
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "create", "update", "list", "put"]
  - apiGroups: [""]
    resources: ["configmaps", "endpoints", "services", "services/status", "nodes"]
    verbs: ["list","get","watch", "update"]
...

kubectl apply -f kube-vip.yaml -n default
```

Create a test aplication with a LoadBalancer type service.

```
kubectl apply -f https://raw.githubusercontent.com/inlets/inlets-operator/master/contrib/nginx-sample-deployment.yaml -n default
kubectl expose deployment nginx-1 --port=80 --type=LoadBalancer -n default
```

As you can see in the logs it creates the the VIP `172.17.8.202`

```bash
kubectl logs kube-vip-cluster-79f767d56f-jkc7f

time="2021-04-14T16:57:58Z" level=info msg="Beginning cluster membership, namespace [default], lock name [plunder-lock], id [k8s-node3]"
I0414 16:57:58.813913       1 leaderelection.go:242] attempting to acquire leader lease  default/plunder-lock...
I0414 16:57:58.857158       1 leaderelection.go:252] successfully acquired lease default/plunder-lock
time="2021-04-14T16:57:58Z" level=info msg="Beginning watching Kubernetes configMap [plndr]"
time="2021-04-14T16:57:58Z" level=debug msg="ConfigMap [plndr] has been Created or modified"
time="2021-04-14T16:57:58Z" level=debug msg="Found 0 services defined in ConfigMap"
time="2021-04-14T16:57:58Z" level=debug msg="[STARTING] Service Sync"
time="2021-04-14T16:57:58Z" level=debug msg="[COMPLETE] Service Sync"
time="2021-04-14T16:59:55Z" level=debug msg="ConfigMap [plndr] has been Created or modified"
time="2021-04-14T16:59:55Z" level=debug msg="Found 1 services defined in ConfigMap"
time="2021-04-14T16:59:55Z" level=debug msg="[STARTING] Service Sync"
time="2021-04-14T16:59:55Z" level=info msg="New VIP [172.17.8.202] for [nginx-1/7676b532-3004-4d41-9282-90765bc98d40] "
time="2021-04-14T16:59:55Z" level=info msg="Starting kube-vip as a single node cluster"
time="2021-04-14T16:59:55Z" level=info msg="This node is assuming leadership of the cluster"
time="2021-04-14T16:59:55Z" level=info msg="Starting TCP Load Balancer for service [172.17.8.202:80]"
time="2021-04-14T16:59:55Z" level=info msg="Load Balancer [nginx-1-load-balancer] started"
time="2021-04-14T16:59:55Z" level=info msg="Broadcasting ARP update for 172.17.8.202 (08:00:27:93:fe:45) via enp0s8"
time="2021-04-14T16:59:55Z" level=info msg="Started Load Balancer and Virtual IP"
time="2021-04-14T16:59:55Z" level=debug msg="[COMPLETE] Service Sync"
time="2021-04-14T16:59:55Z" level=info msg="Beginning watching Kubernetes Endpoints for service [nginx-1]"
time="2021-04-14T16:59:55Z" level=debug msg="Endpoints for service [nginx-1] have  been Created or modified"
time="2021-04-14T16:59:55Z" level=debug msg="Load-Balancer updated with [1] backends"
-> Address: 10.42.1.2:80 
```

It is working on `172.17.8.202` but not perfect because it didn't write back to the api server so the service remain in the "pending" state.

```bash
kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.43.0.1       <none>        443/TCP        83m
nginx-1      LoadBalancer   10.43.126.209   <pending>     80:31904/TCP   46m
```

kube-vip is good solution for High-Availability but for a network LoadBalancer you better to use MetalLB.

### Update: 0.3.8

With kube-vip version 0.3.8 the network LoadBalancer is working:

```bash
kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.43.0.1       <none>        443/TCP        87m
nginx-1      LoadBalancer   10.43.126.209   172.17.8.201     80:31904/TCP   42m
```

