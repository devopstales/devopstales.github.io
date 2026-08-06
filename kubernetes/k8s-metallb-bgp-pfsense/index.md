---
title: Self-hosted Load Balancer for bare metal Kubernetes
url: https://devopstales.github.io/kubernetes/k8s-metallb-bgp-pfsense/
date: 2020-08-18
keywords: Kubernetes, k8s, Load Balancer, BGP, kubernetes deployment, kubernetes ingress, MetalLB
---


In this tutorial I will show you how to install  Metal LB load balancer in BGP mode for Kubernetes.
<!--more-->

{{< content "/filedir/kubernetes.html" >}}

### Metallb
Metallb is a fantastic bare metal-targeted operator for powering LoadBalancer types of services. It can work in two modes: Layer 2 and Border Gateway Protocol (BGP) mode. In layer 2 mode, one of the nodes advertises the load balanced IP (VIP) via ARP. This mode has two limitations: all the traffic goes through a single node VIP potentially limiting the bandwidth. The second limitation is a very slow failover. Detecting unhealthy nodes is a notoriously slow operation in Kubernetes which can take several minutes (5-10 minutes, which can be decreased with the node-problem-detector DaemonSet).

In BGP mode, Metallb advertises the VIP through BGP. It requires a BGP compatible router ath the network wher the kubernetes cluster is created.

 I found that the layer 2 mode of Metallb is not a practical solution for production scenarios as it is typically not acceptable to have failover-induced downtimes in the order of minutes.

### How does the full setup look like?

For this Demo I will use a pfsense in virtualbox and tree vm for kubernetes in the same host-only network.

| vm | nic | ip | mode |
|----|-----|----|------|
| pfsense01 | em1 | 192.168.0.200 | bridged |
| pfsense01 | em2 | 172.17.9.200 | host-only |
| k8sm01 | enp0s8 | 172.17.9.10 | host-only |
| k8sm02 | enp0s8 | 172.17.9.11 | host-only |
| k8sm02 |enp0s8 | 172.17.9.12 | host-only |

### Issues with Calico
Simple BGP config with Calico don’t require anything special. However, if you are using Calico’s <a href="https://docs.projectcalico.org/networking/bgp">external BGP peering capability</a> to advertise your cluster prefixes over BGP, and also want to use BGP in MetalLB, you will need to jump through some hoops.

### Configuring pfSense and OpenBGPD
First we need to install OpenBGPD pcakage on pfSense. Go to `System > Package Manager > Available Packages` Then select `OpenBGPD` and Install it.

There is otger BGP compatible pckages in pfSense so make  sure you DO NOT have the Quagga_OSPF or FRR packages installed. They directly conflict with each other.

No we need to configure BGP. Ther is a nice UI but I will use the Raw config for simplicity. Go to `Services > OpenBGPD > Raw config`

```yaml
# This file was created by the package manager. Do not edit!

AS 64512
fib-update yes
listen on 172.17.9.200
router-id 172.17.9.200
network 10.25.0.0/22

neighbor 172.17.9.10 {
	remote-as 64513
    announce all
	descr "k8sm01"
}

neighbor 172.17.9.11 {
	remote-as 64513
    announce all
	descr "k8sm02"
}

neighbor 172.17.9.12 {
	remote-as 64513
    announce all
	descr "k8sm03"
}
```

* `AS 64512` - Thi is the Autonomous System Number of pfsense
* `listen on ` -   This is the address that OpenBGPD should listen to BGP requests on. I highly recommend setting this to the same as the `router-id` IP address.
* `network` - This is the network what you will use for advertise Load Balanced services.
* `neighbor $(kubernetes_worker_node_ip)` - The ip of the Kubernetes host
* `remote-as 64513` - Thi is the Autonomous System Number of the neighbors. Same for all Kubernetes Node.
* `announce all` - We need our nodes to be able to announce to the router their service IP addresses.

### Configuring pfSense and Quagga_OSPF
If you prefer to use `Quagga_OSPF` Go to `System > Package Manager > Available Packages` Then select `Quagga_OSPF` and Install it.

To configure BGP with `Quagga_OSPF`. Go to `Services > Quagga OSPFd > Raw config` Edit `SAVED bgpd.conf` the `save`.

```bash
router bgp 64512
 bgp router-id 172.17.9.200
 neighbor 172.17.9.10 remote-as 64513
 neighbor 172.17.9.11 remote-as 64513
 neighbor 172.17.9.12 remote-as 64513
  network 10.25.0.0/22
```

### Deploy MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

Make a BGP config for MetalLB

```yaml
nano bgpconfig.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    peers:
    - peer-address: 172.17.9.200
      peer-asn: 64512
      my-asn: 64513
    address-pools:
    - name: default
      protocol: bgp
      addresses:
      - 10.25.0.10-10.25.3.250
```

### Demo Time
Let’s create a demo application for testing.

```yaml
nano test.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx
  namespace: default
spec:
  selector:
    matchLabels:
      run: test-nginx
  replicas: 3
  template:
    metadata:
      labels:
        run: test-nginx
    spec:
      containers:
      - name: test-nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: test-nginx
  namespace: default
  labels:
    run: test-nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: test-nginx
```

```bash
kubectl apply -f bgpconfig.yaml
kubectl apply -f test.yaml
```

After a few moments, you can run this command to get the IP address:

```bash
$ kubectl describe service test-nginx | grep "LoadBalancer Ingress"
LoadBalancer Ingress:     10.25.0.11
```

Let's check the address in a browser. If pfSense is you default gateway it will work perfectly, but in my demo enviroment I need to create a route to pfSense for this network on my host machine:

```bash
sudo route add -net 10.25.0.0/22 gw 172.17.9.200
```

```bash
route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.0.1     0.0.0.0         UG    600    0        0 wlan0
10.25.0.0       172.17.9.200    255.255.252.0   UG    0      0        0 vboxnet7
172.17.9.0      0.0.0.0         255.255.255.0   U     0      0        0 vboxnet7
```

![Example image](/img/include/pfsense-bgp-kubernetes.webp)

