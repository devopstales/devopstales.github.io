---
title: Install AKS Cluster
url: https://devopstales.github.io/cloud/aks/
date: 2020-11-07
keywords: Azure, AKS, streaming
---


In this pos I will show you how you can create an AKS (Azure Kubernetes Service) Cluster.
<!--more-->

{{< content "/filedir/aks.html" >}}

### What is Azure Kubernetes Service?
Azure Kubernetes Service (AKS) is a managed Kubernetes service that lets you quickly deploy and manage clusters.

A Kubernetes cluster is divided into two components:

* Control plane nodes provide the core Kubernetes services and orchestration of application workloads.
* Nodes run your application workloads.

![Example image](/img/include/aks-control-plane-and-nodes.webp)

When you create an AKS cluster, a control plane is automatically created and configured. This control plane is provided as a managed Azure resource abstracted from the user. There's no cost for the control plane, only the nodes that are part of the AKS cluster. The control plane and its resources reside only on the region where you created the cluster.

### How To Create Azure Kubernetes Cluster
There are 2 ways to deploy an Azure Kubernetes Cluster, which are using:

* Azure Portal
* Azure CLI

I prefer to use the cli because it is easier to reproduce.

#### Install Azure CLI
```
# OSX
brew update && brew install azure-cli

# Yum
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc

sudo sh -c 'echo -e "[azure-cli]
name=Azure CLI
baseurl=https://packages.microsoft.com/yumrepos/azure-cli
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/azure-cli.repo'

sudo yum install azure-cli

# apt
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

sudo apt-get update
sudo apt-get install ca-certificates curl apt-transport-https lsb-release gnupg

curl -sL https://packages.microsoft.com/keys/microsoft.asc |
    gpg --dearmor |
    sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null

AZ_REPO=$(lsb_release -cs)
echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $AZ_REPO main" |
    sudo tee /etc/apt/sources.list.d/azure-cli.list

sudo apt-get update
sudo apt-get install azure-cli
```

#### Set the subscription
```
az login
az account list
az account set --subscription <SUBSCRIPTION_ID>
```

#### Creating an Azure Resource Group
```
az group create --location <REGION_NAME> --name <RESOURCE_GROUP_NAME>
```

#### Create AKS Cluster
```
# get available kubernetes versions
az aks get-versions --location <REGION_NAME>

az aks create --resource-group <RESOURCE_GROUP_NAME> \
--name <AKS_CLUSTER_NAME> \
--node-count 3 \
--generate-ssh-keys \
--kubernetes-version 1.17.0
```

### Uses availability zones for AKS Cluster
AKS clusters that are deployed using availability zones can distribute nodes across multiple zones within a single region. For example, a cluster in the East US 2 region can create nodes in all three availability zones in East US 2. This distribution of AKS cluster resources improves cluster availability as they're resilient to failure of a specific zone.

KÉP: https://docs.microsoft.com/en-us/azure/aks/media/availability-zones/aks-availability-zones.png

```
# get available kubernetes versions
az aks get-versions --location <REGION_NAME>

az aks create --resource-group <RESOURCE_GROUP_NAME> \
--name <AKS_CLUSTER_NAME> \
--node-count 3 \
--zones 1 2 3 \
--vm-set-type VirtualMachineScaleSets \
--load-balancer-sku standard \
--generate-ssh-keys \
--kubernetes-version 1.17.0
```


#### Get Kubeconfig Congig of the Cluster
```
az aks get-credentials --name <AKS_CLUSTER_NAME> \
 --resource-group <RESOURCE_GROUP_NAME>

kubectl get nodes
```

