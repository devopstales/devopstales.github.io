---
title: Install Grafana Loki with Helm3
url: https://devopstales.github.io/kubernetes/helm3-loki/
date: 2020-01-03
keywords: grafana, dashboard, loki, kubernetes, logging, helm
---


Helm is a template based package management system for kubernetes applications.
<!--more-->

### Wath is new in Helm3

The most important change in Helm3, tiller was removed completely. Tiller was the server component (rinninf in a pod on the Kubernetes cluster) for helm's cli. When Helm 2 was developed, Kubernetes did not yet have role-based access control (RBAC) therefore to achieve mentioned goal, Helm had to take care of that itself. After Kubernetes 1.6 RBAC is enabled by default so you had to create a serviceaccount for tiller. With Tiller gone, Helm permissions are now simply evaluated using kubeconfig file.

Tiller was also used as a central hub for Helm release information and for maintaining the Helm state. In Helm 3 the same information are fetched directly from Kubernetes API Server and Charts are rendered client-side.

Helm 2 stored the informations of the releases in configmaps now in Helm 3 that is stored in secrets for better security.

### Install Chart with Helm3
The removal of Tiller means you didn't need a `helm init` for initializing the tiller.

```
helm repo add loki https://grafana.github.io/loki/charts
helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm repo update

kubectl create namespace loki-stackhtop

helm upgrade --install loki --namespace=loki-stack loki/loki-stack
elm3 upgrade --install grafana --namespace=loki-stack stable/grafana
```

Namespaces are important now. `helm ls` won’t show anything, we have to specify the namespace with it:
```
helm -n loki-stack ls
```

### Access Grafana Interface
```
kubectl get secret -n loki-stack grafana -o jsonpath="{.data.admin-password}" | base64 --decode
kubectl port-forward -n loki-stack service/grafana 3000:80

# go to localhost:3000
# add loki datasource URL http://loki:3100 and press Save & Test
```

![Example image](/img/include/loki_1.webp)

![Example image](/img/include/loki_2.webp)

![Example image](/img/include/loki_3.webp)

![Example image](/img/include/loki_4.webp)

