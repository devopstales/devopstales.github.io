---
title: Kubernetes with external Ingress Controller with Haproxy and BGP
url: https://devopstales.github.io/kubernetes/k8s-dmz-bgp/
date: 2024-03-03
keywords: kubernetes deployment, haproxy, Kubernetes, ingress controller, cilium, bgp
---


In this post I will show you how to nstall HAProxy Igress Controller on a separate VM instad of running it in the Kubernetes cluster as a pod. For this I  will use cilium BGP pod CIDR export option.

<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

At the company I working wor we head to create a kubernetes cluster with the DMZ network integration, but I didn't want to place the kubernetes cluster to the DMZ network. So I started to find a way to place only the Ingress Controller to a separate node and place only that to the DMZ. I found thet ther is an option with the HAProxy Igress Controller called external mode wher you can run the Ingress Controller not in a pod in the cluster but on a separate node.

### Ciliium config

Enable BGP in cilium:

```yaml
nano cilium-helm-values.yaml
...
bgpControlPlane:
  enabled: true
...
```

```bash
helm upgrade cilium cilium/cilium -f cilium-helm-values.yaml -n kube-system

# test ciliumBGP CRD
k api-resources | grep -i ciliumBGP

# If not exists delete the operator pod
k delete pod -n kube-system cilium-operator-768959858c-zjjnc

# now exists
k api-resources | grep -i ciliumBGP
ciliumbgppeeringpolicies    bgpp    cilium.io/v2alpha1                     false        CiliumBGPPeeringPolicy
```

> I will use 3179 port as BGP port besause cilium ha no privilage for standerd 179

Annotate all the nodes to use port 3179

```bash
kubectl annotate node m102 cilium.io/bgp-virtual-router.65002="local-port=3179"
kubectl annotate node m102 cilium.io/bgp-virtual-router.65002="local-port=3179"
kubectl annotate node m102 cilium.io/bgp-virtual-router.65002="local-port=3179"
```

Ceate the Cilium BGP pearing policy:

```yaml
nano cilium-bgp-policy.yaml
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPPeeringPolicy
metadata:
 name: bgp-peering-policy
spec:
 virtualRouters:
 - localASN: 65001
   exportPodCIDR: true
   neighbors:
    - peerAddress: '192.168.56.13/32'
      peerASN: 65001
```

> When `localASN` and `peerASN` are the same, iBGP peering is used. When `localASN` and `peerASN` are different, eBGP peering is used.
>
> 1) '''External Border Gateway Protocol (EBGP)''': EBGP is used between autonomous systems. It is used and implemented at the edge or border router that provides inter-connectivity for two or more autonomous system. It functions as the protocol responsible for interconnection of networks from different organizations or the Internet.
> 2) '''Internal Border Gateway Protocol (IBGP)''': IBGP is used inside the autonomous systems. It is used to provide information to your internal routers. It requires all the devices in same autonomous systems to form full mesh topology or either of Route reflectors and Confederation for prefix learning.

```bash
kubectl apply -f cilium-bgp-policy.yaml
```

### Install BIRD

On the External Ingress COntroller's VM we vill install the BRD BGP client to estabish connection betwean the VM and the K8S INternal network:

```bash
sudo apt-get install software-properties-common
sudo add-apt-repository -y ppa:cz.nic-labs/bird
sudo apt update
sudo apt install bird
```

```bash
nano /etc/bird/bird.conf

router id 192.168.56.15;
log syslog all;debug protocols all;

# cluster nodes (add all nodes)
protocol bgp m101 {
  local 192.168.56.15 as 65001;
  neighbor 192.168.56.141 port 3179 as 65001;
  import all;
  export all;
  multihop;
  graceful restart;
}
protocol bgp m102 {
  local 192.168.56.15 as 65001;
  neighbor 192.168.56.142 port 3179 as 65001;
  import all;
  export none;
  multihop;
  graceful restart;
}
protocol bgp m103 {
  local 192.168.56.15 as 65001;
  neighbor 192.168.56.143 port 3179 as 65001;
  import all;
  export none;
  multihop;
  graceful restart;
}

# Inserts routes into the kernel routing table
protocol kernel {
   scan time 20;
   import all;
   export all;
   persist;
}

# Gets information about network interfaces from the kernel
protocol device {
   scan time 56;
}
```

```bash
sudo systemctl enable bird
sudo systemctl restart bird
```

### Test the BGP resoults

On cilium:

```bash
cilium bgp peers
Node        Local AS   Peer AS   Peer Address     Session State   Uptime   Family         Received   Advertised
m101        65001      65001     192.168.56.15    established     6s       ipv4/unicast   0          2
                                                                           ipv6/unicast   0          0
m102        65001      65001     192.168.56.15    established     2s       ipv4/unicast   0          2
                                                                           ipv6/unicast   0          0
m103        65001      65001     192.168.56.15    established     3s       ipv4/unicast   0          2
                                                                           ipv6/unicast   0          0
```

On VM:

```bash
sudo birdc show protocols
BIRD 1.6.8 ready.
name     proto    table    state  since       info
m101     BGP      master   up     11:37:24    Established
m102     BGP      master   up     11:37:28    Established
m103     BGP      master   up     11:37:27    Established
kernel1  Kernel   master   up     11:37:23
device1  Device   master   up     11:37:23

birdc show route
BIRD 1.6.8 ready.
10.244.3.0/24      via 192.168.56.141 on enp0s8 [m102 11:37:29] * (100/-) [i]
10.244.4.0/24      via 192.168.56.142 on enp0s8 [m101 11:37:25] * (100/-) [i]
10.244.5.0/24      via 192.168.56.143 on enp0s8 [m103 11:37:28] * (100/-) [i]

route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         _gateway        0.0.0.0         UG    100    0        0 enp0s3
10.0.2.0        0.0.0.0         255.255.255.0   U     0      0        0 enp0s3
_gateway        0.0.0.0         255.255.255.255 UH    100    0        0 enp0s3
10.244.3.0      192.168.56.141  255.255.255.0   UG    0      0        0 enp0s8
10.244.4.0      192.168.56.142  255.255.255.0   UG    0      0        0 enp0s8
10.244.5.0      192.168.56.143  255.255.255.0   UG    0      0        0 enp0s8
192.168.56.0    0.0.0.0         255.255.255.0   U     0      0        0 enp0s8
```

### Install the ingress controller outside of your cluster

Copy the `/etc/kubernetes/admin.conf` kubeconfig file from the control plane server to this server and store it in the root user’s home directory. The ingress controller will use this to connect to the Kubernetes API.

```bash
sudo mkdir -p /root/.kube
sudo cp admin.conf /root/.kube/config
sudo chown -R root:root /root/.kube
```

```bash
kubectl create configmap -n default haproxy-kubernetes-ingress
```

HAProxy Kubernetes Ingress Controller is compatible with a specific version of HAProxy. Install the HAProxy package on your Linux distribution based on the table below. For Ubuntu and Debian, follow the install steps at [haproxy.debian.net](https://haproxy.debian.net/)

| Ingress controller version | Compatible HAProxy version |
| -------------------------- | -------------------------- |
| 1.11 | 2.8 |
| 1.10 | 2.7 |
| 1.9 | 2.6 |
| 1.8 | 2.5 |
| 1.7 | 2.4 |

In my case I use Ingress controller version `1.11` and HAProxy version `2.8`:

```bash
curl https://haproxy.debian.net/bernat.debian.org.gpg \
      | gpg --dearmor > /usr/share/keyrings/haproxy.debian.net.gpg
echo deb "[signed-by=/usr/share/keyrings/haproxy.debian.net.gpg]" \
      http://haproxy.debian.net bookworm-backports-2.8 main \
      > /etc/apt/sources.list.d/haproxy.list

apt-get update
apt-get install haproxy=2.8.\*
```

Stop and disable the HAProxy service.

```bash
sudo systemctl stop haproxy
sudo systemctl disable haproxy
```

Call the `setcap` command to allow HAProxy to bind to ports 80 and 443:

```bash
sudo setcap cap_net_bind_service=+ep /usr/sbin/haproxy
```

Download the ingress controller from the project’s [GitHub Releases page](https://github.com/haproxytech/kubernetes-ingress/releases/)
Extract it and then copy it to the /usr/local/bin directory.

```bash
wget https://github.com/haproxytech/kubernetes-ingress/releases/download/v1.11.3/haproxy-ingress-controller_1.11.3_Linux_x86_64.tar.gz

tar -xzvf haproxy-ingress-controller_1.11.3_Linux_x86_64.tar.gz
sudo cp ./haproxy-ingress-controller /usr/local/bin/
```

Create the file `/lib/systemd/system/haproxy-ingress.service` and add the following to it:

```bash
cat << EOF > /lib/systemd/system/haproxy-ingress.service
[Unit]
Description="HAProxy Kubernetes Ingress Controller"
Documentation=https://www.haproxy.com/
Requires=network-online.target
After=network-online.target
[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/local/bin/haproxy-ingress-controller --external --default-ssl-certificate=ingress-system/default-cert --configmap=default/haproxy-kubernetes-ingress --program=/usr/sbin/haproxy --disable-ipv6 --ipv4-bind-address=0.0.0.0 --http-bind-port=80 --https-bind-port=443  --ingress.class=ingress-public
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGTERM
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```

Enable and start the service.

```bash
sudo systemctl enable haproxy-ingress
sudo systemctl start haproxy-ingress
```

### Demo Aapp

```bash
nano demo-app.yaml
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  annotations:
    ingressclass.kubernetes.io/is-default-class: "false"
  labels:
    app.kubernetes.io/instance: ingress-public
  name: ingress-public
spec:
  controller: haproxy.org/ingress-controller/ingress-public
---
apiVersion: apps/v1
kind: Deployment
metadata:
 labels:
   run: app
 name: app
spec:
 replicas: 5
 selector:
   matchLabels:
     run: app
 template:
   metadata:
     labels:
       run: app
   spec:
     containers:
     - name: app
       image: jmalloc/echo-server
       ports:
       - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
 labels:
   run: app
 name: app
spec:
 selector:
   run: app
 ports:
 - name: port-1
   port: 80
   protocol: TCP
   targetPort: 8080

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: test-ingress
 namespace: default
spec:
  ingressClassName: ingress-public
  rules:
  - host: "example.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app
            port:
              number: 80
```

