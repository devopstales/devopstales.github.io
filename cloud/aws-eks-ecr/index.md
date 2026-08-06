---
title: Elastic Container Registry Integration with EKS
url: https://devopstales.github.io/cloud/aws-eks-ecr/
date: 2022-02-26
keywords: kubernetes deployment, AWS, EKS, Elastic Kubernetes Registry, eksctl, ECR, Elastic Container Registry
---


In this pos I will show you how you can integrate your Elastic Container Registry with EKS.
<!--more-->


```bash
aws ecr create-repository --repository-name aws-ecr-kubenginx --region us-east-1
```

Build end push image
```bash
# Build image with <ECR-REPOSITORY-URI>:<TAG>
docker build -t 180789647333.dkr.ecr.us-east-1.amazonaws.com/aws-ecr-kubenginx:1.0.0 .

# Get Login Password
# aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <ECR-REPOSITORY-URI>
aws ecr get-login-password --region us-east-1 | \
docker login --username AWS --password-stdin 180789647333.dkr.ecr.us-east-1.amazonaws.com/aws-ecr-kubenginx

# Push the Docker Image
docker push <ECR-REPOSITORY-URI>:<TAG>
docker push 180789647333.dkr.ecr.us-east-1.amazonaws.com/aws-ecr-kubenginx:1.0.0
```

Verify ECR Access to EKS Worker Nodes

* Go to Services -> EC2 -> Running Instances > Select a Worker Node -> Description Tab
* Click on value in `IAM Role` field Role name
* In IAM on that `specific role`, verify `permissions` tab
* Policy with name `AmazonEC2ContainerRegistryReadOnly`, `AmazonEC2ContainerRegistryPowerUser` should be associated

Use ECR image with Amazon EKS

```yaml
#01-ECR-Nginx-Deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
name: kubeapp-ecr
labels:
   app: kubeapp-ecr
spec:
replicas: 2
selector:
   matchLabels:
      app: kubeapp-ecr
template:
   metadata:
      labels:
      app: kubeapp-ecr
   spec:
      containers:
      - name: kubeapp-ecr
         image: 180789647333.dkr.ecr.us-east-1.amazonaws.com/aws-ecr-kubenginx:1.0.0
         resources:
            requests:
            memory: "128Mi"
            cpu: "500m"
            limits:
            memory: "256Mi"
            cpu: "1000m"
         ports:
            - containerPort: 80
```
