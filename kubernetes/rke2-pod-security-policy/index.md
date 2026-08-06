---
title: RKE2 Pod Security Policy
url: https://devopstales.github.io/kubernetes/rke2-pod-security-policy/
date: 2020-12-10
keywords: kubernetes deployment, kubernetes vs docker, Kubernetes Secuity, best Practices, rancher, Pod Security Policy
---


In this post I will show you how you can use Pod Security Policys in RKE2.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

### What is a Pod Security Policy?
A Pod Security Policy is a cluster-level resource that controls security sensitive aspects of the pod specification. RBAC Controlls the usable Kubernetes objects for a user but nt the conditions of a specific ofject like allow run as root or not in a container.  PSP objects define a set of conditions that a pod must run with in order to be accepted into the system, as well as defaults for their related fields. PodSecurityPolicy is an optional admission controller that is enabled by default through the API, thus policies can be deployed without the PSP admission plugin enabled.

### PSP examples using RKE2
RKE2 can be ran with or without the `profile: cis-1.5` configuration parameter. This will cause it to apply different PodSecurityPolicies (PSPs) at start-up. If running with the `cis-1.5` profile, RKE2 will apply a restrictive policy called `global-restricted-psp` to all namespaces except `kube-system`. The `kube-system` namespace needs a less restrictive policy named `system-unrestricted-psp` in order to launch critical components.

The policies are outlined below.
```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: global-restricted-psp
spec:
  privileged: false                # CIS - 5.2.1
  allowPrivilegeEscalation: false  # CIS - 5.2.5
  requiredDropCapabilities:        # CIS - 5.2.7/8/9
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  hostNetwork: false               # CIS - 5.2.4
  hostIPC: false                   # CIS - 5.2.3
  hostPID: false                   # CIS - 5.2.2
  runAsUser:
    rule: 'MustRunAsNonRoot'       # CIS - 5.2.6
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  readOnlyRootFilesystem: false
```

This PSP disables `privileged` and `allowPrivilegeEscalation` and force tu run conatiners with UserID and GroupID betwean 1-65535 threat means you cannot run containers wit UserID/GroupID 0 what is root.

The "system unrestricted policy" is applied. See below.
```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: system-unrestricted-psp
spec:
  privileged: true
  allowPrivilegeEscalation: true
  allowedCapabilities:
  - '*'
  volumes:
  - '*'
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  hostIPC: true
  hostPID: true
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

### Test PSP
Is I try to deploy a Deployment with a container running as root it will fail.
```
kubectl get deploy,rs,pod
# output
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/alpine-test   0/1     0            0           67s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.extensions/alpine-test-85c976cdd   1         0         0       67s
```

What happened?
```
kubectl describe replicaset.extensions/alpine-test-85c976cdd | tail -n3
# output
 Type     Reason        Age                    From                   Message
  ----     ------        ----                   ----                   -------
  Warning  FailedCreate  114s (x16 over 4m38s)  replicaset-controller  Error creating: pods "alpine-test-85c976cdd-" is forbidden: unable to validate against any pod security policy: []
```

### Custom PSP
I usually create a restricted rule with allowing the root user in the cobtainer because some operator's container still use it.
```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: allow-root-psp
spec:
  privileged: false                # CIS - 5.2.1
  allowPrivilegeEscalation: false  # CIS - 5.2.5
  requiredDropCapabilities:        # CIS - 5.2.7/8/9
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  hostNetwork: false               # CIS - 5.2.4
  hostIPC: false                   # CIS - 5.2.3
  hostPID: false                   # CIS - 5.2.2
  runAsUser:
    rule: 'MustRunAsNonRoot'       # CIS - 5.2.6
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  readOnlyRootFilesystem: false
```

To use this PSP we need to create a `ClusterRole`.
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: allow-root-psp-role
rules:
- apiGroups:
  - policy
  resourceNames:
  - allow-root-psp
  resources:
  - podsecuritypolicies
  verbs:
  - use
  ```

