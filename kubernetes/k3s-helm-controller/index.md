---
title: Install and use rancher helm-controller
url: https://devopstales.github.io/kubernetes/k3s-helm-controller/
date: 2020-09-20
keywords: kubernetes deployment, Rancher, K3S, K8S, Kubernetes, Helm3, Helm2
---


K3S comes with a Helm operator called Helm Controller. Helm Controller defines a new HelmChart custom resource definition, or CRD, for managing Helm charts.

<!--more-->

{{< content "/filedir/k3s.html" >}}


### Install helm-controller K8S
Thanks to Francisco Bobadilla who created RBAC for Rancher's helm-controller we can deploy this solution to a stander K8S cluster.

```
kubectl apply -f https://raw.githubusercontent.com/iotops/helm-controller/master/manifests/deploy-namespaced.yaml
```

```
kubectl api-resources --api-group=helm.cattle.io
NAME         SHORTNAMES   APIGROUP         NAMESPACED   KIND
helmcharts                helm.cattle.io   true         HelmChart
```

### Deploy Nginx ingress Controller
```
nano ginx-ingress.yaml
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: nginx-ingress
  namespace: helm-controller
spec:
  chart: stable/nginx-ingress
  targetNamespace: nginx-ingress
  valuesContent: |-
    rbac:
      create: "true"
    controller:

      kind: DaemonSet
      hostNetwork: "true"
      daemonset:
        useHostPort: "true"
      service:
        type: "NodePort"
```

On a K3S cluster the `namespace:` must be `kube-system` because k3S deploys the helm-controller to that namespace.

### Auto-Deploying Manifests
Any file found in `/var/lib/rancher/k3s/server/manifests` will automatically be deployed to Kubernetes in a manner similar to `kubectl apply`.

```
cp nginx-ingress.yaml /var/lib/rancher/k3s/server/manifests/
```

