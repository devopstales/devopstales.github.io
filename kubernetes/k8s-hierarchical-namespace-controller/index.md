---
title: Hierarchical namespace controller
url: https://devopstales.github.io/kubernetes/k8s-hierarchical-namespace-controller/
date: 2024-10-25
keywords: Kubernetes, K8S, namespace, Hierarchical namespace controller, quota
---



In this Post I will show you how to install Hierarchical namespace controller (HNC) on k8s.
<!--more-->

### What is Hierarchical namespace controller?

Hierarchical namespace controller is a controller that allows you to create a hierarchy between namespace objects.

### Installing

You can install or upgrade HNC on your cluster using the following commands (admin privileges required):

```bash
# Select the latest version of HNC
HNC_VERSION=v1.1.0

# Select the variant of HNC you like. Other than 'default', options include:
# 'hrq': Like default, but with hierarchical quotas.
# 'ha': Like default, but with two deployments: one single-pod for the controller, and one three-pod for the webhooks
# 'default-cm': Like default, but without the built-in cert rotator, and with support for cert-manager
HNC_VARIANT=default

# Install HNC. Afterwards, wait up to 30s for HNC to refresh the certificates on its webhooks.
kubectl apply -f https://github.com/kubernetes-sigs/hierarchical-namespaces/releases/download/${HNC_VERSION}/${HNC_VARIANT}.yaml 

# Install kubectl plugin with krew
kubectl krew install hns
```

### Basic functionality

> Demonstrates: setting parent-child relationships, subnamespace creation, RBAC propagation, hierarchy modification.

```bash
kubectl create namespace acme-org

kubectl create namespace team-a
kubectl create namespace service-1
```

By default, there's no relationship between these namespaces. Let's make `acme-org` the parent of `team-a`, and `team-a` the parent of `service-1`.

```bash
# Make acme-org the parent of team-a
kubectl hns set team-a --parent acme-org

# This won't work, will be rejected since it would cause a cycle. Try it!
kubectl hns set acme-org --parent team-a

# Make team-a the parent of service-1
kubectl hns set service-1 --parent team-a

# Display the hierarchy
kubectl hns tree acme-org

# Output:
acme-org
└── team-a
     └── service-1
```

### Subnamespaces

Imagine you have a team called `team-b` under the org called `acme-org`. You are the admin of the namespace `team-b` and have no cluster-wide permissions to create namespaces, but you do have a `RoleBinding` in `team-b` that gives you permission to create a `SubnamespaceAnchor` object in that namespace.

Now you would like to create three `subnamespaces` for services owned by your team, say `service-1`, `service-2` and `service-3`:

```bash
kubectl hns create service-1 -n team-b
kubectl hns create service-2 -n team-b
kubectl hns create service-3 -n team-b
kubectl hns tree team-a

# Expected output:
team-b
├── [s] service-1
├── [s] service-2
└── [s] service-3

[s] Indicates subnamespace
```

As with regular namespaces ("full namespaces," in HNC terms), subnamespaces must have unique names. 

### Hierarchical RBAC

```bash
kubectl -n team-a create role team-a-sre --verb=update --resource=deployments
kubectl -n team-a create rolebinding team-a-sres --role team-a-sre --serviceaccount=team-a:default

kubectl -n acme-org create role org-sre --verb=update --resource=deployments
kubectl -n acme-org create rolebinding org-sres --role org-sre --serviceaccount=acme-org:default
```

Now, if we check `service-1`, we'll see that the rolebindings from the ancestor namespaces have been propagated to the child namespace.

```bash
kubectl -n service-1 describe roles
# Output: you should see two roles, one propagated from acme-org and the other
# from team-a.

kubectl -n service-1 get rolebindings
# Output: similarly, you should see two propagated rolebindings.
```

```bash
kubectl describe rolebinding org-sres -n team-a
# Output:
Name:         sres
Labels:       hnc.x-k8s.io/inheritedFrom=acme-org  # inserted by HNC
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  admin
Subjects: ...
```

The controller keeps the RBAC objects in sync with the current hierarchy. 

### Hierarchical network policy

```bash
kubectl hns create service-2 -n team-a
kubectl hns tree acme-org

acme-org
├── team-a
│   ├── service-1
│   └── [s] service-2
└── [s] team-b
```

```bash
kubectl run s2 -n service-2 --image=nginxinc/nginx-unprivileged --restart=Never --expose --port 8080

# Verify that it's running:
kubectl get service,pod -o wide -n service-2

# To test that the service is accessible from workloads in different namespaces, start a client pod in service-1 and confirm that the service is reachable.
kubectl run client -n service-1 -it --image=alpine --restart=Never --rm -- sh
wget -qO- --timeout 2 http://s2.service-2:8080
```

Now we'll create a default network policy that blocks any ingress from other namespaces:

```yaml
kubectl apply -f - << EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: acme-org
spec:
  podSelector:
    matchLabels:
  ingress:
  - from:
    - podSelector: {}
EOF
```

Now let's ensure this policy can be propagated to its descendants.

```bash
kubectl hns config set-resource networkpolicies --group networking.k8s.io --mode Propagate

# And verify it got propagated:
kubectl get netpol --all-namespaces | grep deny

# Expected output:
acme-org    deny-from-other-namespaces   <none>         37s
service-1   deny-from-other-namespaces   <none>         0s
service-2   deny-from-other-namespaces   <none>         0s
team-a      deny-from-other-namespaces   <none>         0s
team-b      deny-from-other-namespaces   <none>         0s
```

If network policies have been correctly enabled on your cluster, we'll now see that we can no longer access `service-2` from a client in `service-1`:

```bash
kubectl run client -n service-1 -it --image=alpine --restart=Never --rm -- sh
wget -qO- --timeout 2 http://s2.service-2:8080
# Observe timeout
```

But in this example, we'd like all service from a specific team to be able to interact with each other, so we'll create a second network policy as shown below that will allow all namespaces within team-a to be able to communicate with each other.

```yaml
kubectl apply -f - << EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-team-a
  namespace: team-a
spec:
  podSelector:
    matchLabels:
  ingress:
  - from:
    - namespaceSelector:
        matchExpressions:
          - key: 'team-a.tree.hnc.x-k8s.io/depth'
            operator: Exists
EOF
```

### Hierarchical resource quotas (HRQs)

Let's assume you own acme-org from the previous example:

```bash
acme-org
├── [s] team-a
└── [s] team-b
```

If you create a `HierarchicalResourceQuota` in namespace `acme-org`, the sum of all `subnamespaces` resources can't surpass the `HRQ`.

Create the HRQ:

```yaml
kubectl apply -f - << EOF
apiVersion: hnc.x-k8s.io/v1alpha2
kind: HierarchicalResourceQuota
metadata:
  name: acme-org-hrq
  namespace: acme-org
spec:
  hard:
    services: 1
EOF
```

```bash
kubectl create service clusterip team-a-svc --clusterip=None -n team-a

# Now, when you try to create a Service in namespace team-b:
kubectl create service clusterip team-b-svc --clusterip=None -n team-b

# You get an error:
Error from server (Forbidden):
  error when creating "STDIN":
    admission webhood "resourcesquotasstatus.hnc.x-k8s.io" denied the request:
      exceeded hierarchical quota in namespace "acme-org":
        "acme-org-hrq", requested: services=1, used: services=1, limited: services=1
```


