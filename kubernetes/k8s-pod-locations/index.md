---
title: Influencing Kubernetes Scheduler Decisions
url: https://devopstales.github.io/kubernetes/k8s-pod-locations/
date: 2023-04-10
keywords: Kubernetes Scheduler, Taints, Tolerations, Node Affinity, Pod Affinity, topologySpreadConstraints
---


In this post I will show you how you can influence the Kubernetes Scheduler where to schedule a pod.
<!--more-->

### Kubernetes Scheduler

When the Kubernetes Scheduler schedule a pod it examines each node for whether or not it can host the Pod. The scheduler uses the following equation to calculate the available memory on a given node: `Usable memory = available memory - reserved memory`

The reserved memory refers to:

* Memory used by Kubernetes daemons like kubelet, containerd (or another container runtime).
* Memory is used by the node’s operating system. For example, kernel daemons.

If you are following the best practices you are declaring the amount of CPU and memory your Pods require through requests and limits.

### Influencing the Scheduling Process

You have multiple ways to influence the scheduler. In the simplest way is to force a Pod to run on one - and only one - node by specifying its name in the `.spec.nodeName`.

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: nginx
spec:
 containers:
 - name: nginx
   image: nginx
 nodeName: app-prod01
```

### Taints and Tolerations

Suppose we didn’t want any pods to run on a specific node. You might need to do this for a variety of reasons. Whatever the particular reason, we need a way to ensure our pods are not placed on a certain node. That’s where a taint comes in.

When a node is `tainted`, no Pod can be scheduled to it unless the Pod `tolerates` the taint. You can taint a node with a command like this:

```bash
kubectl taint nodes [node name] [key=value]:NoSchedule

kubectl taint nodes worker-01 locked=true:NoSchedule
```

The definition for a Pod that has the necessary toleration to get scheduled on the tainted node look like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: mypod
spec:
 containers:
 - name: mycontainer
   image: nginx
 tolerations:
 - key: "locked"
   operator: "Equal"
   value: "true"
   effect: "NoSchedule"
```

### Node Affinity

Node Affinity gives you more flexible way to choose a node by allowing you to define hard and soft node-requirements. The hard requirements must be matched on the node to be selected, but the soft requirements allows you to add more weight to nodes with specific labels. The mos basic example for this scenario to choose a node with ssd for your database instance:

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: db
spec:
 affinity:
   nodeAffinity:
     requiredDuringSchedulingIgnoredDuringExecution:
       nodeSelectorTerms:
       - matchExpressions:
         - key: disk-type
           operator: In
           values:
           - ssd
     preferredDuringSchedulingIgnoredDuringExecution:
     - weight: 1
       preference:
         matchExpressions:
         - key: zone
           operator: In
           values:
           - zone1
           - zone2
 containers:
 - name: db
   image: mysql
```

The `requiredDuringSchedulingIgnoredDuringExecution` is the hard requirement and the `preferredDuringSchedulingIgnoredDuringExecution` is the soft requirement. You can add Affinity not just for a nod but the pods themselves.

### Pod Affinity

You can yous hard (`requiredDuringSchedulingIgnoredDuringExecution`) and soft (`preferredDuringSchedulingIgnoredDuringExecution`) requirements for the pods too:

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: middleware
spec:
 affinity:
   podAffinity:
     requiredDuringSchedulingIgnoredDuringExecution:
     - labelSelector:
         matchExpressions:
         - key: role
           operator: In
           values:
           - frontend
       topologyKey: kubernetes.io/hostname
   podAntiAffinity:
     preferredDuringSchedulingIgnoredDuringExecution:
     - weight: 100
       podAffinityTerm:
         labelSelector:
           matchExpressions:
           - key: role
             operator: In
             values:
             - auth
         topologyKey: kubernetes.io/hostname
 containers:
 - name: middleware
   image: redis
```

You may have noticed that both the hard and soft requirements have the IgnoredDuringExecution suffix. It means that after the scheduling decision has been made, the scheduler will not attempt to change already-placed Pods even if the conditions changed.

For example if your application has multiple replicas and you did't want to schedule two pod from this app to the same host:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 5
  template:
    metadata:
      labels:                                            
        app: nginx                                   
    spec:
      affinity:
        podAntiAffinity:                                 
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:                               
              matchLabels:                               
                app: nginx        
      container:
        image: nginx:latest
```

### Pod Topology Spread Constraints

You can use topology spread constraints to control how Pods are spread across your cluster among failure-domains such as regions, zones, nodes.

Suppose you have a 4-node cluster with the following labels:

```bash
NAME    STATUS   ROLES    AGE     VERSION   LABELS
node1   Ready    <none>   4m26s   v1.16.0   node=node1,zone=zoneA
node2   Ready    <none>   3m58s   v1.16.0   node=node2,zone=zoneA
node3   Ready    <none>   3m17s   v1.16.0   node=node3,zone=zoneB
node4   Ready    <none>   2m43s   v1.16.0   node=node4,zone=zoneB
```

You can define one or multiple `topologySpreadConstraint` to instruct the kube-scheduler how to place each incoming Pod in relation to the existing Pods across your cluster. If we want an incoming Pod to be evenly spread with existing Pods across zones, the spec can be something like this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: nginx
spec:
  replicas: 8
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: nginx
      containers:
      - name: nginx
        image: nginx:latest
```

Besides the usual deployment specification, we have additionally defined `TopologySpreadConstraints` as such:

* **maxSkew**: 1 — distribute pods in an absolute even manner
* **topologyKey**: `kubernetes.io/hostname` —use the hostname as topology domain
* **whenUnsatisfiable**: ScheduleAnyway — always schedule pods even if it can’t satisfy even distribution of pods
* **labelSelector** —only act on Pods that match this selector


You can use 2 `TopologySpreadConstraints` to control the Pods spreading on both zone and node:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: nginx
spec:
  replicas: 8
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: nginx
        - maxSkew: 1
          topologyKey: node
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: nginx
      containers:
      - name: nginx
        image: nginx:latest
```

Ofcourse, you can use the default `kubernetes.io/hostname` label for `topologyKey`.

