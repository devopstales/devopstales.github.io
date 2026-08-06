---
title: Use multus to separate metwork trafics
url: https://devopstales.github.io/kubernetes/multus-calico/
date: 2022-01-15
keywords: Kubernetes, k8s, multus, calico, kubernetes deployment, cni
---


In this post I will show you how you can use Multus CNI and Calico to create Kubernetes pods in different networks.

<!--more-->

### Install default network

> The kubernetes cluster is installed with `kubeadm` and `--pod-network-cidr=10.244.0.0/16` option

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

nano custom-resources.yaml
---
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    bgp: Disabled
    # linuxDataplane: BPF
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

kubectl apply -f custom-resources.yaml
```

### Install Secondary network

```bash
ip -d link show vxlan.calico
9: vxlan.calico: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default
    link/ether 66:2f:69:dc:0c:cc brd ff:ff:ff:ff:ff:ff promiscuity 0 minmtu 68 maxmtu 65535
    vxlan id 4096 local 192.168.200.10 dev enp0s9 srcport 0 0 dstport 4789 nolearning ttl auto ageing 300 udpcsum noudp6zerocsumtx noudp6zerocsumrx addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535

ip a show vxlan.calico
9: vxlan.calico: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether 66:2f:69:dc:0c:cc brd ff:ff:ff:ff:ff:ff
    inet 10.250.58.192/32 scope global vxlan.calico
       valid_lft forever preferred_lft forever
    inet6 fe80::642f:69ff:fedc:ccc/64 scope link
       valid_lft forever preferred_lft forever
```

### Deploy multus

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml

kubectl get pods --all-namespaces | grep -i multus
```

```bash
ls -laF /opt/cni/bin
total 291839
drwxr-xr-x 1 root root       13 Jul 22 13:39 ./
drwxr-xr-x 1 root root        3 May 25 23:22 ../
-rwxr-xr-x 1 root root  3954360 Jul 22 12:57 bandwidth*
-rwsr-xr-x 1 root root 62013206 Jul 22 12:57 calico*
-rwsr-xr-x 1 root root 62013206 Jul 22 12:57 calico-ipam*
-rwxr-xr-x 1 root root  2342446 Jul 22 12:57 flannel*
-rwxr-xr-x 1 root root  3421269 Jul 22 12:57 host-local*
-rwsr-xr-x 1 root root 62013206 Jul 22 12:57 install*
-rwxr-xr-x 1 root root  3484706 Jul 22 12:57 loopback*
-rwxr-xr-x 1 root root 46926142 Jul 22 13:39 multus*
-rwxr-xr-x 1 root root 39561855 Jul 22 13:05 multus-shim*
-rwxr-xr-x 1 root root  3913894 Jul 22 12:57 portmap*
-rwxr-xr-x 1 root root  4372265 May 25 23:21 ptp*
-rwxr-xr-x 1 root root  3641046 Jul 22 12:57 tuning*
```

```bash
cat /etc/cni/net.d/00-multus.conf | jq
```

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz
tar -xzf cni-plugins-linux-amd64-v1.2.0.tgz -C /opt/cni/bin ./host-local ./ipvlan ./macvlan
```

check multus config find calico as the default config.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: 22-calico
spec: 
  config: '{
    "cniVersion": "0.3.1",
    "type": "calico",
    "log_level": "info",
    "datastore_type": "kubernetes",
    "ipam": {
      "type": "host-local",
      "subnet": "172.22.0.0/16"
    },
    "policy": {
      "type": "k8s"
    },
    "kubernetes": {
      "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
    }
  }'
EOF

cat <<EOF | kubectl apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: 26-calico
spec: 
  config: '{
    "cniVersion": "0.3.1",
    "type": "calico",
    "log_level": "info",
    "datastore_type": "kubernetes",
    "ipam": {
      "type": "host-local",
      "subnet": "172.26.0.0/16"
    },
    "policy": {
      "type": "k8s"
    },
    "kubernetes": {
      "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
    }
  }'
EOF
```

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: net-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: default/26-calico
spec:
  containers:
  - name: netshoot-pod
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
---
apiVersion: v1
kind: Pod
metadata:
  name: net-pod2
  annotations:
    k8s.v1.cni.cncf.io/networks: default/22-calico
spec:
  containers:
  - name: netshoot-pod
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
EOF
```

The `default/22-calico` means that the `22-calico` NetworkAttachmentDefinition from the `default` namespace.

```yaml
kubectl describe pods net-pod
...
Annotations:      cni.projectcalico.org/containerID: 3e73ddecf916a31d93685bbd6bbc0663ffebfcbb6417bbc71717e1512e83c315
                  cni.projectcalico.org/podIP: 172.26.0.2/32
                  cni.projectcalico.org/podIPs: 172.26.0.2/32
                  k8s.v1.cni.cncf.io/network-status:
                    [{
                        "name": "k8s-pod-network",
                        "ips": [
                            "10.244.39.202"
                        ],
                        "default": true,
                        "dns": {}
                    },{
                        "name": "default/26-calico",
                        "ips": [
                            "172.26.0.2"
                        ],
                        "dns": {}
                    }]
                  k8s.v1.cni.cncf.io/networks: default/26-calico
Status:           Running
IP:               10.244.39.202
IPs:
  IP:  10.244.39.202
...
Events:
  Type    Reason          Age   From               Message
  ----    ------          ----  ----               -------
  Normal  Scheduled       11m   default-scheduler  Successfully assigned default/net-pod to test-worker2
  Normal  AddedInterface  11m   multus             Add eth0 [10.244.39.202/32] from k8s-pod-network
  Normal  AddedInterface  11m   multus             Add net1 [172.26.0.2/32] from default/26-calico
  Normal  Pulling         11m   kubelet            Pulling image "nicolaka/netshoot"
  Normal  Pulled          11m   kubelet            Successfully pulled image "nicolaka/netshoot" in 2.123400585s (2.12353655s including waiting)
  Normal  Created         11m   kubelet            Created container netshoot-pod
  Normal  Started         11m   kubelet            Started container netshoot-pod
```

```bash
kubectl exec -it net-pod -- ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if66: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether fa:26:21:4b:3c:94 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.26.0.4/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f826:21ff:fe4b:3c94/64 scope link
       valid_lft forever preferred_lft forever

kubectl exec -it net-pod2 -- ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if67: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether fe:16:f1:9d:5e:40 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.22.0.6/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::fc16:f1ff:fe9d:5e40/64 scope link
       valid_lft forever preferred_lft forever
```

```bash
kubectl exec -it net-pod -- ping -c 1 172.22.0.6
PING 172.22.0.6 (172.22.0.6) 56(84) bytes of data.
64 bytes from 172.22.0.6: icmp_seq=1 ttl=63 time=0.099 ms

--- 172.22.0.6 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.099/0.099/0.099/0.000 ms
```
