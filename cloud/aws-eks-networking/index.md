---
title: AWS EKS Network Solutions
url: https://devopstales.github.io/cloud/aws-eks-networking/
date: 2022-02-10
keywords: kubernetes deployment, CNI, CNI, Container Network Interface, calico, AWS, EKS, VPC CNI
---


In this post I will analyse the available CNI plugins for Amazon Elastic Kubernetes Service.
<!--more-->

Cloud providers like AWS, Google are in the market with their managed Kubernetes offering, which is one of the easiest ways to getting started with the Kubernetes journey. The cloud provider manage most of the components, but you still get a chance to choose your own CNI. Different providers offers different CNI plugins. AWS give you two option Calico or VPC CNI.

### VPC CNI

Amazon VPC CNI Plugin assigns a private IPv4 or IPv6 address from your VPC to each pod. By default, the number of pods can you run on a worker is based on the number of IP addresses assigned to Elastic network interfaces and the number of network interfaces attached to your Amazon EC2 node. From the v1.19.0 support higher pod density per node by allow to assign /28 (16 IP addresses) prefixes, instead of assigning individual IP addresses to network interfaces. PAMD will derive a (/32) IP from these prefixes for pod IP allocation.

To add or update the add-on on your cluster, see [Managing the Amazon VPC CNI add-on](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html). If you want to update the version in the existing cluster, run the below command.

```bash
eksctl update addon \
 --name vpc-cni \
 --version <1.9.x-eksbuild.y> \
 --cluster <my-cluster> \
 --force

kubectl set env daemonset aws-node \
 -n kube-system ENABLE_PREFIX_DELEGATION=true
```

To use this CNI you need an [Amazon EC2 Nitro instances](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html) for node groups in the cluster.

```bash
eksctl create nodegroup \
  --cluster <my-cluster> \
  --region <us-east-1> \
  --name <my-nodegroup> \
  --node-type <m5.large> \
  --managed \
  --max-pods-per-node <110> -- Count from the max pods calculator
```

### Calico
We can yous any other standerd CNI plugin, but the other suppertid CNI in EKS is Caliso. The huge advantage of Calico is the ability to create Network policies. Network policies are similar to AWS security groups in that you can create network ingress and egress rules. Instead of assigning instances to a security group, you assign network policies to pods using pod selectors and labels. 

If you go with your own CNI, AWS will not be supporting any issues related to networking, as they don’t manage it. They are right at their end but definitely something you should watch out for.

The Other Problem is the communication with the api server from the pods. For every EKS cluster, AWS launches the control plane in a VPC which is managed by AWS. At the time of creating the control plane AWS creates a route table entry for your VPC CNI CIDR to the ENI of the control plane. This is how the API server is able to communicate to all the Nodes and pods in the cluster, but if we decide to use our own CNI, like calico, we change the POD CIDR, and now the API server has no clue on how to route the traffic to the pod IP. 

![AWS VPN CNI Routing](/img/include/aws-eks-vpc-cni.webp)

Some situation It can be a problem fi your pods can't communicate with te api server. This means you cannot use admission webhooks, operators or Service Mesh. On the other hand it can be a security feature too. 
