---
title: Install k8s with IPVS mode
url: https://devopstales.github.io/kubernetes/k8s-ipvs/
date: 2020-10-13
keywords: kubernetes deployment, kubernetes vs docker, kube-proxy, IPVS
---


In this post I will show you how to install kubernetes with kube-proxy IPVS mode.
<!--more-->

{{< content "/filedir/kubernetes.html" >}}

### Wthat is kube-proxy
kube-proxy is a key component of any Kubernetes deployment.  Its role is to load-balance traffic to the pods. It listens to all the service requests coming through from kubernetes and creates entries in iptables for each of these service IPs to achieve proper routing to the pod. So kube-proxy adds iptables ruleset for each new service defined. As the number of services grow, this list is going to be huge. This potentially impact the performance because the iptables processing is sequential and wit every new line the list goes longer and longer. Kubernetes's solution for this problem was IPVS.

### What is IPVS?
IPVS (IP Virtual Server) is built on top of the Netfilter and implements transport-layer load balancing as part of the Linux kernel. It runs on a host and acts as a load balancer in front of a cluster of real servers. IPVS can direct requests for TCP- and UDP-based services to the real servers, and make services of the real servers appear as virtual services on a single IP address. Therefore, IPVS naturally supports Kubernetes Service. IPVS mode provides greater scale and performance vs iptables mode.

Installing Kubernetes with IPVS kube-proxy mode is a little bit hard because there in no built in option for theat in kubeadm. So we have two option. Createt a custom kubeadm.yaml or edit an installed cluster.

### Install Requirements
```
yum install ipset ipvsadm -y

cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

### Createt a custom kubeadm.yaml
```
kubeadm config print init-defaults > kubeadm.yaml

nano kubeadm.yaml
...
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
```

### Edit running cluster
```
kubectl edit configmap kube-proxy -n kube-system
...
mode: ipvs
```

```
kubectl get po -n kube-system
kubectl delete po -n kube-system <pod-name>
```

```
kubectl logs [kube-proxy pod] | grep "Using ipvs Proxier"
```

### Test IPVS mode is running
```
ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 1.1.1.101:6443               Masq    1      0          0
TCP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0
  -> 10.244.2.8:53                Masq    1      0          0
```

