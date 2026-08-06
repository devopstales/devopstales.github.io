---
title: Install Nginx Ingress Controller to AKS with application routing add-on
url: https://devopstales.github.io/cloud/aks-ingress-controller-v2/
date: 2024-07-15
keywords: Azure, AKS, streaming, aks-app-routing-operator, ingress, ingress controller, private, public
---


In this pos I will show you how you can install Inginx Ingress Controller to a AKS (Azure Kubernetes Service) Cluster with aks-app-routing-operator.
<!--more-->

{{< content "/filedir/aks.html" >}}

Azure created it own operator for managing Nginx Ingress Controller creation called `aks-app-routing-operator`. It is integrated with Azure so you didn't need to install it with helm, and can be used with Other Azure services for DNS management and certificate generation.

### Get AKS credentials

```bash
az login
az aks get-credentials --resource-group test-cluster --name test-cluster
kubectl get nodes
```

### Enable aks-app-routing-operator on AKS Cluster

Enable on New Cluster

```bash
az aks create \
--resource-group <ResourceGroupName> \
--name <ClusterName> --location <Location> \
--enable-app-routing --generate-ssh-keys
```

Enable on Existing Cluster

```bash
# az aks approuting enable
az aks approuting enable --resource-group <ResourceGroupName> --name <ClusterName>

# az aks enable-addons
az aks enable-addons --resource-group <ResourceGroupName> \
--name <ClusterName> --addons http_application_routing
```

### Use Ingress Controller with a public ip address

The application routing add-on creates an Ingress class on the cluster named webapprouting.kubernetes.azure.com. When you create an Ingress object with this class, it activates the add-on.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: aks-helloworld
  namespace: hello-web-app-routing
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
  - host: <Hostname>
    http:
      paths:
      - backend:
          service:
            name: aks-helloworld
            port:
              number: 80
        path: /
        pathType: Prefix
```

```bash
kubectl apply -f ingress.yaml -n hello-web-app-routing

kubectl get ingress -n hello-web-app-routing

NAME             CLASS                                HOSTS               ADDRESS       PORTS     AGE
aks-helloworld   webapprouting.kubernetes.azure.com   myapp.contoso.com   20.51.92.19   80, 443   4m
```

### Configure custom ingress controller configurations

When you enable the application routing add-on with NGINX, it creates an ingress controller called `default` in the `app-routing-namespace` configured with a public facing Azure load balancer. That ingress controller uses an ingress class name of `webapprouting.kubernetes.azure.com`.

Create another public facing NGINX ingress controller

```yaml
apiVersion: approuting.kubernetes.azure.com/v1alpha1
kind: NginxIngressController
metadata:
  name: nginx-public
spec:
  ingressClassName: nginx-public
  controllerNamePrefix: nginx-public
```

```bash
kubectl apply -f nginx-public-controller.yaml
```

### Create an internal NGINX ingress controller with a private IP address

```yaml
apiVersion: approuting.kubernetes.azure.com/v1alpha1
kind: NginxIngressController
metadata:
  name: nginx-internal
spec:
  ingressClassName: nginx-internal
  controllerNamePrefix: nginx-internal
  loadBalancerAnnotations: 
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

```bash
kubectl apply -f nginx-internal-controller.yaml
```

### Create an NGINX ingress controller with a static IP address

To create an NGINX ingress controller with a static IP address on the Azure Load Balancer:

```bash
az network public-ip create \
--location <REGION_NAME> \
--resource-group <RESOURCE_GROUP_NAME> \
--name <IP_NAME> --sku Standard \
--allocation-method static \
--query publicIp.ipAddress -o tsv

# 51.105.230.165
```

Ensure the cluster identity used by the AKS cluster has delegated permissions to the public IP's resource group:

```bash
CLIENT_ID=$(az aks show --name <ClusterName> --resource-group <ClusterResourceGroup> --query identity.principalId -o tsv)
RG_SCOPE=$(az group show --name myNetworkResourceGroup --query id -o tsv)
az role assignment create --assignee ${CLIENT_ID} --role "Network Contributor" --scope ${RG_SCOPE}
```

Create an ingress controller:

```yaml
apiVersion: approuting.kubernetes.azure.com/v1alpha1
kind: NginxIngressController
metadata:
  name: nginx-static
spec:
  ingressClassName: nginx-static
  controllerNamePrefix: nginx-static
  loadBalancerAnnotations: 
    service.beta.kubernetes.io/azure-pip-name: "<IP_NAME>"
    service.beta.kubernetes.io/azure-load-balancer-resource-group: "<RESOURCE_GROUP_NAME>"
```

```bash
kubectl apply -f nginx-staticip-controller.yaml

kubectl get nginxingresscontroller -n <IngressControllerName>

NAME           INGRESSCLASS   CONTROLLERNAMEPREFIX   AVAILABLE
nginx-public   nginx-public   nginx                  True

kubectl get nginxingresscontroller -n <IngressControllerName> \
-o jsonpath='{range .items[*].status.conditions[*]}{.lastTransitionTime}{"\t"}{.status}{"\t"}{.type}{"\t"}{.message}{"\n"}{end}'

2023-11-29T19:59:24Z    True    IngressClassReady       Ingress Class is up-to-date
2023-11-29T19:59:50Z    True    Available               Controller Deployment has minimum availability and IngressClass is up-to-date
2023-11-29T19:59:50Z    True    ControllerAvailable     Controller Deployment is available
2023-11-29T19:59:25Z    True    Progressing             Controller Deployment has successfully progressed
```

Use the ingress controller in an ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: aks-helloworld
  namespace: hello-web-app-routing
spec:
  ingressClassName: <IngressClassName>
  rules:
  - host: <Hostname>
    http:
      paths:
      - backend:
          service:
            name: aks-helloworld
            port:
              number: 80
        path: /
        pathType: Prefix
```

