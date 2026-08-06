---
title: Use Cilium BGP integration with OPNsense
url: https://devopstales.github.io/kubernetes/cilium-opnsense-bgp-v2/
date: 2023-01-25
keywords: Kubernetes, k8s, Load Balancer, BGP, kubernetes deployment, kubernetes ingress, Cilium, OPNsense
---


In this tutorial I will show you how to install Cilium with BGP integration for Kubernetes.
<!--more-->

In the upcoming release, version 1.13, Cilium will include a functionality which allow to advertise routes to Service IPs via BGP, so we didn't need to install MetalLB.

> Cilium version 1.13 is still in pre-release phase.

### How does the full setup look like?

For this Demo I will use a pfsense in virtualbox and tree vm for kubernetes in the same host-only network.

| vm | nic | ip | mode |
|----|-----|----|------|
| opnsense01 | em1 | 192.168.0.200 | bridged |
| opnsense01 | em2 | 172.17.9.200 | host-only |
| k8sm01 | enp0s8 | 172.17.9.10 | host-only |
| k8sm02 | enp0s8 | 172.17.9.11 | host-only |
| k8sm02 |enp0s8 | 172.17.9.12 | host-only |


### Install BGP to OPNsense

Go to `System > Firmware > Plugins` and install `os-frr`

### Configure BGP on OPNsense

Go tp `Routing > General` and enable enable the plugin. Next go to `Routing > BGP` and enble, then add AS Number.

![Enable BGP](/img/include/cilium-opnsense-bgp1.webp)

### Configure Neighbor

Go to  `Routing > BGP` the switch to the `Neighbor` tab and add the following three neighbors.

![Neighbor](/img/include/cilium-opnsense-bgp2.webp)

![Neighbor](/img/include/cilium-opnsense-bgp3.webp)

![Neighbor](/img/include/cilium-opnsense-bgp4.webp)

### Cilium configuration
BGP support is enabled by providing the BGP configuration via a ConfigMap and by setting a few Helm values. Otherwise, BGP is disabled by default.

```bash
helm install cilium cilium/cilium \
--set bgp.enabled=true
```

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: opensense
spec:
  virtualRouters:
    - localASN: 64513
      neighbors:
        - peerASN: 64512
          peerAddress: 172.17.9.200
---
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: default
spec:
  cidrs:
    - cidr: 10.25.0.10-10.25.3.250
  disabled: false
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

![Demo](/img/include/pfsense-bgp-kubernetes.webp)

