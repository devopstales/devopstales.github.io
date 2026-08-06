---
title: Configure custom SSL and DNS on AKS Ingress Controller
url: https://devopstales.github.io/cloud/custom-ssl-and-dns-on-aks-ingress-controller/
date: 2024-07-19
keywords: Azure, AKS, streaming, ingress, ingress controller, private, public, Azure Private DNS, aks-app-routing-operator, 
---


In this pos I will show you how you can configure custom SSL and DNS on Nginx Ingress Controlle to AKS (Azure Kubernetes Service) Cluster.
<!--more-->

{{< content "/filedir/aks.html" >}}

### Get AKS credentials

```bash
az login
az aks get-credentials --resource-group test-cluster --name test-cluster
kubectl get nodes
```

### Terminate HTTPS traffic with certificates from Azure Key Vault

Create an Azure Key Vault using the az keyvault create command.

```bash
az keyvault create \
--resource-group <ResourceGroupName> \
--location <Location> \
--name <KeyVaultName> \
--enable-rbac-authorization true
```

In this example I will create and export a self-signed SSL certificate, but if you have a valid certificate you can use it directly.

```bash
openssl req -new -x509 -nodes -out aks-ingress-tls.crt \
-keyout aks-ingress-tls.key -subj "/CN=<Hostname>" \
-addext "subjectAltName=DNS:<Hostname>"

openssl pkcs12 -export -in aks-ingress-tls.crt \
-inkey aks-ingress-tls.key -out aks-ingress-tls.pfx
```

Import certificate into Azure Key Vault

```bash
az keyvault certificate import \
--vault-name <KeyVaultName> \
--name <KeyVaultCertificateName> \
--file aks-ingress-tls.pfx \
[--password <certificate password if specified>]
```

### Enable Azure Key Vault integration

As you can see in the ![previous post](/cloud/aks-azure-key-vault-csi/) you can enable Azure Key Vault integration like this:

```bash
KEYVAULTID=$(az keyvault show --name <KeyVaultName> --query "id" --output tsv)

az aks approuting update \
--resource-group <ResourceGroupName> \
--name <ClusterName> \
--enable-kv \
--attach-kv ${KEYVAULTID}
```

### Enable Azure DNS integration

As you can see in the [previous post](/cloud/azure-private-dns-with-aks-ingress-controller/) you can enable Azure Key Vault integration.

### Create the Ingress that uses a host name and a certificate from Azure Key Vault

Get the certificate URI to use in the Ingress from Azure Key Vault using the az keyvault certificate show command.

```bash
az keyvault certificate show \
--vault-name <KeyVaultName> \
--name <KeyVaultCertificateName> \
--query "id" --output tsv
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.azure.com/tls-cert-keyvault-uri: <KeyVaultCertificateUri>
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
  tls:
  - hosts:
    - <Hostname>
    secretName: keyvault-<your ingress name>
```

```bash
kubectl apply -f ingress.yaml -n hello-web-app-routing

kubectl get ingress -n hello-web-app-routing

NAME             CLASS                                HOSTS               ADDRESS       PORTS     AGE
aks-helloworld   webapprouting.kubernetes.azure.com   myapp.contoso.com   20.51.92.19   80, 443   4m
```

