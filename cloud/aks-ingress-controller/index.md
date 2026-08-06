---
title: Install Nginx Ingress Controller to AKS
url: https://devopstales.github.io/cloud/aks-ingress-controller/
date: 2020-11-15
keywords: Azure, AKS, streaming, ingress, ingress controller, private, public
---


In this pos I will show you how you can install Nginx Ingress Controlle to AKS (Azure Kubernetes Service) Cluster.
<!--more-->

{{< content "/filedir/aks.html" >}}

### Get AKS credentials

```bash
az login
az aks get-credentials --resource-group test-cluster --name test-cluster
kubectl get nodes
```

### Create ingress with static public ip

```bash
az network public-ip create \
--location <REGION_NAME> \
--resource-group <RESOURCE_GROUP_NAME> \
--name <IP_NAME> --sku Standard \
--allocation-method static \
--query publicIp.ipAddress -o tsv

# 51.105.230.165
```

### Deploy nginx ingress controller with helm

```bash
kubectl create namespace ingress

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm install <INGERSS_NAME> \
ingress-nginx/ingress-nginx \
--namespace ingress \
--set controller.replicaCount=2 \
--set controller.nodeSelector."kubernetes\.io/os"=linux \
--set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
--set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
--set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz \
--set controller.service.loadBalancerIP="51.105.230.165"
```

```bash
kubectl --namespace ingress get services -o wide -w nginx-ingress-controller
kubectl get service -l app=nginx-ingress --namespace ingress
```

### Create an ingress controller to an internal virtual network in

By default, an NGINX ingress controller is created with a dynamic public IP address assignment. A common configuration requirement is to use an internal, private network and IP address. This approach allows you to restrict access to your services to internal users, with no external access. This example assigns 10.240.0.42 to the loadBalancerIP resource.

```bash
helm install nginx-ingress \
ingress-nginx/ingress-nginx \
--namespace ingress \
--set controller.ingressClass=internal \
--set controller.replicaCount=2 \
--set controller.nodeSelector."kubernetes\.io/os"=linux \
--set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
--set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
--set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal"=true \
--set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz \
--set controller.service.loadBalancerIP=10.224.0.42
```

Create an ingress route

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: hello-world-ingress
  namespace: ingress-basic
  annotations:
    kubernetes.io/ingress.class: internal
...
```

