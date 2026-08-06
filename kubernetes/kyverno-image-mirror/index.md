---
title: Automatically change registry in pod definition
url: https://devopstales.github.io/kubernetes/kyverno-image-mirror/
date: 2021-08-16
keywords: kubernetes deployment, kubernetes vs docker, Kyverno, mutating webhook, Admission Controller
---


In this post I will show you how you can automatically change the registry part in deployed pods in Kubernetes.

<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

### ImageSwap Mutating Admission Controller for Kubernetes

The ImageSwap webhook enables you to define one or more mappings to automatically swap image definitions within Kubernetes Pods with a different registry. 

Install ImageSwap:

```bash
$ kubectl apply -f https://raw.githubusercontent.com/phenixblue/imageswap-webhook/v1.4.2/deploy/install.yaml
```

For swapping configuration ImageSwap use a configmap to define what image should change to what:

```yaml
apiVersion: v1
data:
  maps: |
    default:registry.example.com
    #gcr.io: # This is a comment
    gitlab.com:registry.example.com/gitlab
    noswap_wildcards:example.com
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: imageswap-maps
  namespace: imageswap-system
```

Example MAPS Configs:

Disable image swapping for all registries EXCEPT `gcr.io`

```bash
default:
gcr.io:harbor.internal.example.com
```

Enable image swapping for all registries except `gcr.io`

```bash
```

With this, all images will be swapped except those that already match the `harbor.internal.example.com` pattern

```bash
default:harbor.internal.example.com
noswap_wildcards:harbor.internal.example.com
```

### Test ImageSwap

```bash
$ kubectl create ns test1
$ kubectl label ns test1 k8s.twr.io/imageswap=enabled
```

Then deploy a pod:

```bash
kubectl create deployment unsigned-my \
--image=docker.io/devopstales/testimage:unsigned
```

ImageSwap can be disabled on a per workload level by adding the `k8s.twr.io/imageswap` label with a value of `disabled` to the pod template.

### Kyverno

Here is an example of a Kyverno policy that validates that images are only pulled from an allowed list of image registries (based on wildcard patterns):

```yaml
apiVersion : kyverno.io/v1alpha1
kind: Policy
metadata:
  name: check-registries
spec:
  rules:
  - name: check-registries
    resource:
      kinds:
      - Deployment
      - StatefulSet
    validate:
      message: "Registry is not allowed"
      pattern:
        spec:
          template:
            spec:
              containers:
              - name: "*"
                # Check allowed registries
                image: "*/nirmata/* | https://private.registry.io/*"
```

Rather than blocking Pods which come from outside registries, it is also possible to mutate them so the pulls are directed to approved registries.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: replace-image-registry
  annotations:
    policies.kyverno.io/title: Replace Image Registry
    policies.kyverno.io/category: Sample
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod
    policies.kyverno.io/minversion: 1.3.6
    policies.kyverno.io/description: >-
      Rather than blocking Pods which come from outside registries,
      it is also possible to mutate them so the pulls are directed to
      approved registries. In some cases, those registries may function as
      pull-through proxies and can fetch the image if not cached.
      This policy policy mutates all images either
      in the form 'image:tag' or 'registry.corp.com/image:tag' to be prefaced
      with `myregistry.corp.com/`.      
spec:
  background: false
  rules:
    - name: replace-image-registry
      match:
        resources:
          kinds:
          - Pod
      mutate:
        patchStrategicMerge:
          spec:
            containers:
            - (name): "*"
              image: |-
                                {{ regex_replace_all('^[^/]+', '{{@}}', 'myregistry.corp.com') }}
```
