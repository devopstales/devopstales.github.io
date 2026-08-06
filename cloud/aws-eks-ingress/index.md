---
title: Deploy Ingress Controller to EKS cluster with WAF
url: https://devopstales.github.io/cloud/aws-eks-ingress/
date: 2022-02-12
keywords: kubernetes deployment, CNI, Container Network Interface, calico, AWS, EKS, Elastic Kubernetes Registry, VPC CNI, eksctl
---


In this pos I will show you how you can install the AWS Load Balancer Controller on EKS Cluster with WAF protection.
<!--more-->

AWS Load Balancer Controller is a controller to help manage Elastic Load Balancers for a Kubernetes cluster. It satisfies Kubernetes `Ingress` resources by provisioning Application Load Balancers and `Service` resources by provisioning Network Load Balancers.

AWS Elastic Load Balancing Application Load Balancer (ALB) is a popular AWS service that load balances incoming traffic at the application layer (layer 7) across multiple targets, such as Amazon EC2 instances, in multiple Availability Zones. It supports multiple features including: TLS (Transport Layer Security) termination, AWS WAF (Web Application Firewall) integration and integrated access logs, and health checks.

### Deploy the AWS Load Balancer Controller 

```bash
# Let’s start by setting a few environment variables:
export AWS_REGION=eu-central-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
export EKS_CLUSTER_NAME=eks-devopstales

VPC_ID=$(aws eks describe-cluster \
  --name $EKS_CLUSTER_NAME \
  --region $AWS_REGION \
  --query 'cluster.resourcesVpcConfig.vpcId' \
  --output text)

# Associate OIDC provider 
eksctl utils associate-iam-oidc-provider \
  --cluster $EKS_CLUSTER_NAME \
  --region $AWS_REGION \
  --approve

# Download the IAM policy document
curl -S https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.3.0/docs/install/iam_policy.json -o iam-policy.json

# Create an IAM policy
LBC_IAM_POLICY_ARN=$(aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json \
  --query 'Policy.Arn' \
  --output text)

# Create a service account 
eksctl create iamserviceaccount \
  --cluster=$EKS_CLUSTER_NAME \
  --region $AWS_REGION \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --override-existing-serviceaccounts \
  --attach-policy-arn=arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
  
helm repo add eks https://aws.github.io/eks-charts && helm repo update
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=$EKS_CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=$VPC_ID \
  --set region=$AWS_REGION
```

Let’s create a Kubernetes ingress for an app called Yelb. The AWS Load Balancer Controller will associate the ingress with an Application Load Balancer.

```yaml
cat << EOF > yelb-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yelb.app
  namespace: yelb
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: yelb-ui
                port:
                  number: 80
EOF

kubectl apply -f yelb-ingress.yaml
```

### Add a web application firewall to the ingress

The first thing we need to do is create a WAS web ACL. In AWS WAF, a web access control list or a web ACL monitors HTTP(S) requests for one or more AWS resources. These resources can be an Amazon API Gateway, AWS AppSync, Amazon CloudFront, or an Application Load Balancer.

```bash
# Create an AWS WAF web ACL:
WAF_WACL_ARN=$(aws wafv2 create-web-acl \
  --name WAF-FOR-YELB \
  --region $WAF_AWS_REGION \
  --default-action Allow={} \
  --scope REGIONAL \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=YelbWAFAclMetrics \
  --description "WAF Web ACL for Yelb" \
  --query 'Summary.ARN' \
  --output text )

# Store the AWS WAF web ACL’s Id in an environment variable
WAF_WAF_ID=$(aws wafv2 list-web-acls \
  --region $WAF_AWS_REGION \
  --scope REGIONAL \
  --query "WebACLs[?Name=='WAF-for-Yelb'].Id" \
  --output text)
```

```yaml
# Update the ingress and associate this AWS WAF web ACL with the ALB that the ingress uses:
cat << EOF > yelb-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yelb.app
  namespace: yelb
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/wafv2-acl-arn: ${WAF_WACL_ARN}
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: yelb-ui
                port:
                  number: 80
EOF
kubectl apply -f yelb-ingress.yaml
```

### Enable traffic filtering in AWS WAF

We have associated the ALB that our Kubernetes ingress uses with an AWS WAF web ACL Every request that’s handled by our sample application Yelb pods goes through AWS WAF for inspection. The AWS WAF web ACL is currently allowing every request to pass because we haven’t configured any AWS WAF rules.  In order to filter out potentially malicious traffic, we have to specify rules.

AWS WAF Bot Control is a managed rule group that provides visibility and control over common and pervasive bot traffic to web applications. While Bot Control has been optimized to minimize false positives, we recommend that you deploy Bot Control in count mode first and review CloudWatch metrics and AWS WAF logs to ensure that you are not accidentally blocking legitimate traffic. 

Create a rules file and deploy:

```bash
cat << EOF > waf-rules.json 
[
    {
      "Name": "AWS-AWSManagedRulesBotControlRuleSet",
      "Priority": 0,
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesBotControlRuleSet"
        }
      },
      "OverrideAction": {
        "None": {}
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "AWS-AWSManagedRulesBotControlRuleSet"
      }
    }
]
EOF
```

```bash
aws wafv2 update-web-acl \
  --name WAF-FOR-YELB \
  --scope REGIONAL \
  --id $WAF_WAF_ID \
  --default-action Allow={} \
  --lock-token $(aws wafv2 list-web-acls \
    --region $AWS_REGION \
    --scope REGIONAL \
    --query "WebACLs[?Name=='WAF-for-Yelb'].LockToken" \
    --output text) \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=YelbWAFAclMetrics \
  --region $AWS_REGION \
  --rules file://waf-rules.json
```
