---
title: Kubernetes Network Policy
url: https://devopstales.github.io/kubernetes/k8s-networkpolicy/
date: 2021-01-10
keywords: kubernetes deployment, kubernetes vs docker, Kubernetes Secuity, best Practices, NetworkPolicy
---


In this post I will show you how you can use NetworkPolicys in K8S.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

### Network policies
Network policies are Kubernetes resources that allows to control the traffic between pods and/or network endpoints. Most CNI plugins support the implementation of network policies, but if they don't the created `NetworkPolicy` will be ignored.

The most popular CNI plugins with network policy support are:

* Weave
* Calico
* Canal
* Cilium

### Example
A good practice is to define and apply a default NetworkPolicy to deny all incoming traffic to all pods in all application namespaces, then whitelist pods and subnets based on application needs.

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: monitoring
spec:
  policyTypes:
  - Ingress
  - Egress
  podSelector: {}
```

Since this resource defines both policyTypes ingress and egress, but doesn’t define any whitelist rules, it blocks all the pods in the monitoring namespace from communicating with each other. Note that this policy dose not allows connections to port 53 on any IP by default, to facilitate DNS lookups. So we need to whitelist dns. All `NetworkPolicy` is like a firewall rule. To select an aplication you need to use selectors of labels.

```
# create label
kubectl label namespace kube-system networking/namespace=kube-system
```

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all-egress
spec:
  policyTypes:
  - Egress
  podSelector: {}
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          networking/namespace: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
```


To allow connections from the Ingress Controller: 

```
# create label
kubectl label namespace nginx-ingress networking/namespace=ingress
```

```
apiVersion: networking.k8s.io/v1n
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
spec:
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          networking/namespace: ingress
  podSelector: {}
```

We need to create allow rules to define what aplication can communicate with anathor aplication. To match network traffic by combining namespace and pod selectors, you can use a `NetworkPolicy` object similar to the following: 

```
apiVersion: extensions/v1beta1
kind: NetworkPolicy
metadata:
  name: alertmanager-mesh
  namespace: monitoring
spec:
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: prometheus
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - port: 9093
      protocol: tcp
  podSelector:
    matchLabels:
      app: alertmanager
```

Allow inbound tcp to port 9093 from only prometheus to alertmanager

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prometheus
  namespace: monitoring
spec:
  policyTypes:
  - Ingress
  podSelector:
    matchLabels:
      app: prometheus
  ingress:
  - from:
    - podSelector: {}
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090
```

Allow inbound tcp to port 9090 from any source to prometheus.

You can create Rules to allow outboudn trafic from a service to a apps with specific tags. The following policy allows pod outbound traffic to other pods in the same namespace that match the pod selector. In the following example, outbound traffic is allowed only if they go to a pod with label color=red, on port 80.

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-egress-same-namespace
  namespace: default
spec:
  policyTypes:
    - Egress
  podSelector:
    matchLabels:
      color: blue
  egress:
  - to:
    - podSelector:
        matchLabels:
          color: red
    ports:
    - port: 80
```

If You Don’t Know Which Pods Need To Talk To Each Other you can allow all application in a namespace to connect with each other.

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
spec:
  policyTypes:
  - Ingress
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}
```

For services that require egress to resources outside of the cluster, for example, a database whitelist the subnet that the network resource is on. 

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: customer-api-allow-web
  namespace: prod
spec:
  policyTypes:
  - Egress
  podSelector:
    matchLabels:
      app: orders
  egress:
  - ports:
    - port: 3306
    to:
    - ipBlock:
        cidr: 172.16.32.0/27
```
### Calico NetworkPolicy
Calico network policy provides a richer set of policy capabilities than Kubernetes including:

* policy ordering/priority
* deny rules
* Protocols: TCP, UDP, ICMP, SCTP, UDPlite, ICMPv6, protocol numbers (1-255)

Calico network policies apply to endpoints. In Kubernetes, each pod is a Calico endpoint. However, Calico can support other kinds of endpoints. There are two types of Calico endpoints: workload endpoints (such as a Kubernetes pod or OpenStack VM) and host endpoints (an interface or group of interfaces on a host).

```
kind: NetworkPolicy
apiVersion: projectcalico.org/v3
metadata:
  name: allow-egress-same-namespace
  namespace: default
spec:
  selector: color == 'red'
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: color == 'blue'
    destination:
      ports:
        - 80
```

In the following example, incoming TCP traffic to any pods with label color: red is denied if it comes from a pod with color: blue.

```
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-blue
spec:
  selector: color == 'red'
  ingress:
  - action: Deny
    protocol: TCP
    source:
      selector: color == 'blue'
```

Apply network policies in specific order:

```
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: drop-other-ingress
spec:
  order: 20
  ...deny policy rules here...

apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-cluster-internal-ingress
spec:
  order: 10
  ...allow policy rules here...
```

In the following example, incoming TCP traffic to an application is denied, and each connection attempt is logged to syslog:

```
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
Metadata:
  name: allow-tcp-6379
  namespace: production
Spec:
  selector: role == 'database'
  types:
  - Ingress
  - Egress
  ingress:
  - action: Log
    protocol: TCP
    source:
      selector: role == 'frontend'
  - action: Deny
    protocol: TCP
    source:
      selector: role == 'frontend'
```

It is important to enforce separation of containers. As you can see you can create a `NetworkPolicy` for a specific namespace. So don't forget to create the default best practice Policies. In the next post will show you how you can automate the creation of the Default Policies for new namespaces.

