---
title: Install k8s and calico with eBPF mode
url: https://devopstales.github.io/kubernetes/k8s-calico-ebpf/
date: 2020-10-13
keywords: kubernetes deployment, kubernetes vs docker, kube-proxy, eBPF, calico
---


In this post I will show you how to install kubernetes Without kube-proxy using calico with eBPF mode.
<!--more-->

{{< content "/filedir/kubernetes.html" >}}

### Wthat is kube-proxy
kube-proxy is a key component of any Kubernetes deployment.  Its role is to load-balance traffic to the pods. It listens to all the service requests coming through from kubernetes and creates entries in iptables for each of these service IPs to achieve proper routing to the pod. So kube-proxy adds iptables ruleset for each new service defined. As the number of services grow, this list is going to be huge. This potentially impact the performance because the iptables processing is sequential and wit every new line the list goes longer and longer. Kubernetes's solution for this problem was IPVS.

### What is IPVS?
IPVS (IP Virtual Server) is built on top of the Netfilter and implements transport-layer load balancing as part of the Linux kernel. It runs on a host and acts as a load balancer in front of a cluster of real servers. IPVS can direct requests for TCP- and UDP-based services to the real servers, and make services of the real servers appear as virtual services on a single IP address. Therefore, IPVS naturally supports Kubernetes Service. IPVS mode provides greater scale and performance vs iptables mode. However, it comes with some limitations. There is no option to install IPVS mode in kubeadm. <a href="../../cloud/k8s-ipvs/">In the previous post you can see ho you can do install.</a> The other solution is eBPF.

### What is eBPF ?
eBPF is a virtual machine embedded within the Linux kernel. It allows small programs to be loaded into the kernel, and attached to hooks, which are triggered when some event occurs. For example, when a network interface emits a packet.

### Requirements
First you need a supported Linuy Distrubution:
* Ubuntu 20.04.
* Red Hat v8.2 with Linux kernel v4.18.0-193 or above (Red Hat have backported the required features to that build on CentOS 8 but not on 7)
* Pre installed K8s cluster



Verify that your cluster is ready for eBPF mode
```
mount | grep "/sys/fs/bpf"
bpf on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700)
```

### Install Calico With Operator
We need to change the `cidr` in `custom-resources.yaml` to match to our clusters `cdir`.
```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml

wget https://docs.projectcalico.org/manifests/custom-resources.yaml
nano custom-resources.yaml
...
cidr: 10.244.0.0/16


kubectl create -f custom-resources.yaml
watch kubectl get pods -n calico-system
```

### Configure calicoctl
```
export CALICO_DATASTORE_TYPE=kubernetes
export CALICO_KUBECONFIG=~/.kube/config
calicoctl get workloadendpoints
calicoctl get nodes
```

Configure tigera-operator to communicate with kubernetes's api.
```
nano kubernetes-services-endpoint.yaml
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kubernetes-services-endpoint
  namespace: tigera-operator
data:
  KUBERNETES_SERVICE_HOST: "172.17.9.10"
  KUBERNETES_SERVICE_PORT: "6443"

kubectl apply -f  kubernetes-services-endpoint.yaml
kubectl delete pod -n tigera-operator -l k8s-app=tigera-operator
watch kubectl get pods -n calico-system
```

### Change to eBPF mode
First we change kube-proxy's `nodeSelector` to a none existing node to disable `kube-proxy`, the patch calico to run in eBPF mode.
```
kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'
calicoctl patch felixconfiguration default --patch='{"spec": {"bpfKubeProxyIptablesCleanupEnabled": false}}'
calicoctl patch felixconfiguration default --patch='{"spec": {"bpfEnabled": true}}'
calicoctl patch felixconfiguration default --patch='{"spec": {"bpfExternalServiceMode": "DSR"}}'
```

