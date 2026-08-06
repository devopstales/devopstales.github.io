---
title: Kubernetes Egress Gateway with Monzo Egress Operator
url: https://devopstales.github.io/kubernetes/monzo-egress-operator/
date: 2026-04-10
keywords: Monzo egress operator, Kubernetes egress gateway, AWS NAT Gateway, egress CRD, Kubernetes security, egress control, cloud egress
---


Monzo's Egress Operator is an open-source Kubernetes operator that automates the provisioning and management of **AWS NAT Gateways** for Kubernetes egress traffic. This post explores a cloud-native approach to egress gateway management with Kubernetes-native CRDs.

<!--more-->

{{< content "/filedir/egress-gateway-series.html" >}}

## What is Monzo Egress Operator?

Monzo Bank open-sourced their Egress Operator, which **automates the creation and management of AWS NAT Gateways** for Kubernetes clusters. Instead of manually provisioning NAT Gateways, you define egress requirements in Kubernetes CRDs and the operator handles the rest.

### Architecture Overview

```
┌──────────────────────────┐
│ Kubernetes Cluster on AWS│
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│    Egress Operator       │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│     Egress CRD           │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  Creates NAT Gateway     │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ Allocates Elastic IP     │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  Updates Route Tables    │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  Internet Gateway        │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  External Services       │
└──────────────────────────┘
```

### How It Works

1. **Create Egress CRD** - Define egress requirements in Kubernetes
2. **Operator Provisions** - Creates AWS NAT Gateway automatically
3. **Elastic IP Allocation** - Assigns static IP for whitelisting
4. **Route Table Updates** - Configures subnet routing
5. **Automatic Management** - Handles updates and deletion

### Key Features

| Feature | Description |
|---------|-------------|
| **Automated Provisioning** | NAT Gateways created via Kubernetes CRDs |
| **Elastic IP Management** | Static IPs automatically allocated |
| **Route Table Configuration** | Subnet routing automatically updated |
| **High Availability** | Multi-AZ NAT Gateway support |
| **Cost Optimization** | Shares NAT Gateways across namespaces |
| **AWS Native** | Uses AWS NAT Gateway (managed service) |

## Prerequisites

| Component | Version | Notes |
|-----------|---------|-------|
| **Kubernetes** | 1.25+ | EKS recommended |
| **AWS Account** | - | With appropriate IAM permissions |
| **eksctl/terraform** | Latest | For cluster setup |
| **kubectl** | Latest | Configured with cluster access |
| **IAM Role** | - | With NAT Gateway permissions |

### IAM Permissions Required

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNatGateway",
        "ec2:DeleteNatGateway",
        "ec2:DescribeNatGateways",
        "ec2:AllocateAddress",
        "ec2:ReleaseAddress",
        "ec2:AssociateAddress",
        "ec2:DisassociateAddress",
        "ec2:DescribeAddresses",
        "ec2:DescribeRouteTables",
        "ec2:CreateRoute",
        "ec2:DeleteRoute",
        "ec2:ReplaceRoute",
        "ec2:CreateTags",
        "ec2:DeleteTags"
      ],
      "Resource": "*"
    }
  ]
}
```

## Installation

### Step 1: Install with Helm

```bash
# Add Monzo Helm repository
helm repo add monzo https://monzo.github.io/helm-charts
helm repo update

# Install egress operator
helm install egress-operator monzo/egress-operator \
  --namespace egress-operator \
  --create-namespace \
  --set aws.region=us-east-1 \
  --set aws.clusterName=my-eks-cluster \
  --wait
```

### Step 2: Configure IAM Role

For EKS, create IAM role for service account:

```bash
# Create IAM OIDC provider (if not already done)
eksctl utils associate-iam-oidc-provider \
  --cluster my-eks-cluster \
  --approve

# Create IAM policy
aws iam create-policy \
  --policy-name EgressOperatorPolicy \
  --policy-document file://egress-operator-policy.json

# Create service account with IAM role
eksctl create iamserviceaccount \
  --name egress-operator-controller \
  --namespace egress-operator \
  --cluster my-eks-cluster \
  --attach-policy-arn arn:aws:iam::ACCOUNT_ID:policy/EgressOperatorPolicy \
  --approve \
  --override-existing-serviceaccounts
```

### Step 3: Verify Installation

```bash
# Check operator pods
kubectl get pods -n egress-operator

# Expected output:
# NAME                                  READY   STATUS
# egress-operator-controller-xxxxx      1/1     Running

# Verify CRDs are installed
kubectl get crds | grep egress

# Expected:
# egresses.ec2.monzo.com
# egressnatgateways.ec2.monzo.com
```

## Basic Usage

### Create Simple Egress

```yaml
apiVersion: ec2.monzo.com/v1beta1
kind: Egress
metadata:
  name: production-egress
  namespace: production
spec:
  # AWS region
  region: us-east-1
  
  # VPC ID where NAT Gateway will be created
  vpcId: vpc-12345678
  
  # Subnets for NAT Gateway (one per AZ for HA)
  subnets:
    - subnet-aaa111
    - subnet-bbb222
    - subnet-ccc333
  
  # Tags for AWS resources
  tags:
    Environment: production
    ManagedBy: egress-operator
```

### Apply Egress Resource

```bash
kubectl apply -f egress.yaml
```

### Verify Egress Status

```bash
# Check egress resource
kubectl get egress production-egress -n production

# Expected output:
# NAME                STATUS    ELASTIC_IP        NAT_GATEWAY_ID
# production-egress   Ready     54.123.45.67      nat-0abc123def

# Get detailed status
kubectl describe egress production-egress -n production
```

### Check AWS Resources

```bash
# Verify NAT Gateway in AWS
aws ec2 describe-nat-gateways \
  --filter "Name=tag:Name,Values=production-egress"

# Verify Elastic IP
aws ec2 describe-addresses \
  --filter "Name=tag:Name,Values=production-egress"

# Verify route tables
aws ec2 describe-route-tables \
  --filter "Name=tag:Name,Values=production-egress"
```

## Advanced Configuration

### High Availability (Multi-AZ)

```yaml
apiVersion: ec2.monzo.com/v1beta1
kind: Egress
metadata:
  name: ha-egress
  namespace: production
spec:
  region: us-east-1
  vpcId: vpc-12345678
  
  # Multiple subnets across AZs for HA
  subnets:
    - subnet-aaa111  # us-east-1a
    - subnet-bbb222  # us-east-1b
    - subnet-ccc333  # us-east-1c
  
  # Enable multi-AZ NAT Gateway
  multiAz: true
  
  # Elastic IP configuration
  elasticIp:
    # Let operator allocate new EIP
    autoAllocate: true
    
  tags:
    Environment: production
    HighAvailability: "true"
```

### Bring Your Own Elastic IP

```yaml
apiVersion: ec2.monzo.com/v1beta1
kind: Egress
metadata:
  name: custom-eip-egress
  namespace: production
spec:
  region: us-east-1
  vpcId: vpc-12345678
  subnets:
    - subnet-aaa111
  
  elasticIp:
    # Use existing Elastic IP
    allocationId: eipalloc-12345678
  
  tags:
    Environment: production
```

### Share NAT Gateway Across Namespaces

```yaml
# Create shared NAT Gateway
apiVersion: ec2.monzo.com/v1beta1
kind: Egress
metadata:
  name: shared-egress
  namespace: infrastructure
spec:
  region: us-east-1
  vpcId: vpc-12345678
  subnets:
    - subnet-aaa111
    - subnet-bbb222
  
  # Mark as shareable
  shared: true
  
  tags:
    Environment: production
    Shared: "true"

---
# Reference shared NAT Gateway from other namespaces
apiVersion: ec2.monzo.com/v1beta1
kind: EgressBinding
metadata:
  name: app-egress-binding
  namespace: app-team
spec:
  # Reference shared egress
  egressRef:
    name: shared-egress
    namespace: infrastructure
  
  # Subnets that should use this egress
  subnets:
    - subnet-app111
    - subnet-app222
```

### Cost Optimization with Shared Egress

```yaml
# Infrastructure team manages shared egress
apiVersion: ec2.monzo.com/v1beta1
kind: Egress
metadata:
  name: company-wide-egress
  namespace: network-ops
spec:
  region: us-east-1
  vpcId: vpc-12345678
  subnets:
    - subnet-aaa111
    - subnet-bbb222
  
  # Large NAT Gateway for high throughput
  allocationId: eipalloc-12345678
  
  shared: true
  allowList:
    - namespace: team-a
    - namespace: team-b
    - namespace: team-c
  
  tags:
    CostCenter: network-ops
    Shared: "true"
```

## Integration with Kubernetes Networking

### Subnet Configuration for EKS

```yaml
# EKS cluster with private subnets
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-eks-cluster
  region: us-east-1

vpc:
  id: vpc-12345678
  subnets:
    private:
      us-east-1a:
        id: subnet-aaa111
      us-east-1b:
        id: subnet-bbb222
    public:
      us-east-1a:
        id: subnet-pub111
      us-east-1b:
        id: subnet-pub222

# Egress operator will create NAT Gateway for private subnets
```

### Route Table Management

The operator automatically:

1. **Creates routes** in private subnet route tables
2. **Points to NAT Gateway** as default route (0.0.0.0/0)
3. **Updates on changes** when NAT Gateway is modified

```bash
# Verify route table configuration
aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=production-egress" \
  --query 'RouteTables[].Routes[?DestinationCidrBlock==`0.0.0.0/0`]'

# Expected output:
# [
#   {
#     "DestinationCidrBlock": "0.0.0.0/0",
#     "GatewayId": "nat-0abc123def",
#     "State": "active"
#   }
# ]
```

## Monitoring and Observability

### Operator Metrics

```bash
# Enable Prometheus metrics
helm upgrade egress-operator monzo/egress-operator \
  --namespace egress-operator \
  --set metrics.enabled=true \
  --set metrics.port=8080 \
  --wait

# Access metrics
kubectl port-forward -n egress-operator svc/egress-operator-controller 8080:8080
```

### Key Metrics

| Metric | Description |
|--------|-------------|
| `egress_operator_nat_gateways_total` | Total NAT Gateways managed |
| `egress_operator_elastic_ips_total` | Total Elastic IPs allocated |
| `egress_operator_reconcile_duration_seconds` | Reconciliation duration |
| `egress_operator_reconcile_errors_total` | Reconciliation errors |

### CloudWatch Integration

```yaml
# NAT Gateway metrics automatically sent to CloudWatch
# Key metrics:
# - ActiveConnectionCount
# - BytesInFromDestination
# - BytesOutToDestination
# - ConnectionAttemptCount
# - ConnectionEstablishedCount
# - IdleTimeoutCount
# - PacketsDropCount
# - PacketsInFromDestination
# - PacketsOutToDestination
```

### Grafana Dashboard

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: egress-operator-dashboard
  namespace: monitoring
data:
  egress-operator.json: |
    {
      "dashboard": {
        "title": "Egress Operator - NAT Gateways",
        "panels": [
          {
            "title": "NAT Gateway Status",
            "targets": [
              {
                "expr": "egress_operator_nat_gateways_total"
              }
            ]
          },
          {
            "title": "Elastic IPs",
            "targets": [
              {
                "expr": "egress_operator_elastic_ips_total"
              }
            ]
          },
          {
            "title": "Reconciliation Errors",
            "targets": [
              {
                "expr": "rate(egress_operator_reconcile_errors_total[5m])"
              }
            ]
          }
        ]
      }
    }
```

## Cost Management

### NAT Gateway Pricing (us-east-1)

| Component | Cost |
|-----------|------|
| **NAT Gateway-hour** | $0.045 per hour |
| **Data processed** | $0.045 per GB |
| **Elastic IP (unused)** | $0.005 per hour |

### Cost Optimization Strategies

#### 1. Share NAT Gateways

```yaml
# Instead of one NAT Gateway per namespace
# Create shared NAT Gateway for multiple teams

apiVersion: ec2.monzo.com/v1beta1
kind: Egress
metadata:
  name: shared-production-egress
  namespace: platform
spec:
  region: us-east-1
  vpcId: vpc-12345678
  subnets:
    - subnet-aaa111
    - subnet-bbb222
  
  shared: true
  
  tags:
    CostCenter: platform
    Shared: "true"
```

**Cost Comparison:**

| Approach | NAT Gateways | Monthly Cost |
|----------|--------------|--------------|
| Per namespace (10 teams) | 10 | ~$324/month |
| Shared (1 for all) | 1 | ~$32/month |
| **Savings** | | **~$292/month** |

#### 2. Right-Size NAT Gateways

```yaml
# Monitor traffic and adjust
# Single NAT Gateway handles up to 10 Gbps
# For higher throughput, use multiple

apiVersion: ec2.monzo.com/v1beta1
kind: Egress
metadata:
  name: high-throughput-egress
  namespace: data-team
spec:
  region: us-east-1
  vpcId: vpc-12345678
  
  # Multiple NAT Gateways for high throughput
  subnets:
    - subnet-aaa111  # NAT Gateway 1
    - subnet-bbb222  # NAT Gateway 2
    - subnet-ccc333  # NAT Gateway 3
  
  tags:
    Throughput: high
```

#### 3. Use Tags for Cost Allocation

```yaml
spec:
  tags:
    CostCenter: engineering
    Team: backend
    Environment: production
    Project: api-platform
```

## Troubleshooting

### Issue: NAT Gateway Not Created

```bash
# Check operator logs
kubectl logs -n egress-operator -l app=egress-operator

# Check Egress resource status
kubectl describe egress production-egress -n production

# Look for events
kubectl get events -n production --sort-by='.lastTimestamp'

# Verify IAM permissions
aws sts get-caller-identity

# Check AWS service quotas
aws service-quotas get-service-quota \
  --service-code vpc \
  --quota-code L-FE4A38D6  # NAT Gateways per AZ
```

### Issue: Route Table Not Updated

```bash
# Check route table association
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=subnet-aaa111"

# Verify NAT Gateway is available
aws ec2 describe-nat-gateways \
  --nat-gateway-ids nat-0abc123def

# Check operator reconciliation
kubectl get egress production-egress -n production -o yaml
```

### Issue: Elastic IP Not Allocated

```bash
# Check Elastic IP allocation
aws ec2 describe-addresses \
  --filters "Name=tag:Name,Values=production-egress"

# Check for quota limits
aws service-quotas get-service-quota \
  --service-code vpc \
  --quota-code L-0263D0A3  # Elastic IPs per region

# Verify Egress spec
kubectl get egress production-egress -n production \
  -o jsonpath='{.spec.elasticIp}'
```

### Common Problems and Solutions

| Problem | Cause | Solution |
|---------|-------|----------|
| NAT Gateway stuck in Pending | AWS quota exceeded | Request quota increase |
| Route not created | IAM permission missing | Add ec2:CreateRoute permission |
| EIP not allocated | EIP quota reached | Release unused EIPs or request increase |
| Operator not reconciling | Service account IAM issue | Re-create IAM service account |
| High latency | NAT Gateway overloaded | Add more NAT Gateways (multi-AZ) |

## Security Best Practices

### 1. Restrict Egress by Subnet

```yaml
apiVersion: ec2.monzo.com/v1beta1
kind: Egress
metadata:
  name: restricted-egress
  namespace: secure
spec:
  region: us-east-1
  vpcId: vpc-12345678
  
  # Only specific subnets use this egress
  subnets:
    - subnet-secure111
  
  # Network ACLs should also restrict traffic
  tags:
    Security: restricted
    Compliance: PCI-DSS
```

### 2. Use VPC Flow Logs

```bash
# Enable VPC Flow Logs for audit
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-12345678 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flow-logs
```

### 3. Monitor Egress Traffic

```yaml
# CloudWatch alarm for unusual traffic
aws cloudwatch put-metric-alarm \
  --alarm-name "HighNATEgress" \
  --metric-name BytesOutToDestination \
  --namespace AWS/NatGateway \
  --statistic Sum \
  --period 300 \
  --threshold 1000000000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT:alerts
```

## Comparison with Other Solutions

| Feature | Monzo Egress Operator | Cilium | Antrea | Kube-OVN |
|---------|----------------------|--------|--------|----------|
| **Cloud Provider** | AWS only | Multi-cloud | Multi-cloud | Multi-cloud |
| **NAT Type** | AWS NAT Gateway | Self-managed | Self-managed | Self-managed |
| **Management** | Kubernetes CRD | Cilium CRD | Subnet CRD | Subnet CRD |
| **High Availability** | Multi-AZ NAT | Manual setup | Manual setup | ECMP |
| **Cost** | Pay per NAT Gateway | Free (self-managed) | Free | Free |
| **Operations** | Fully managed by AWS | Self-managed | Self-managed | Self-managed |
| **Scalability** | 10 Gbps per NAT | Limited by node | Limited by node | Limited by node |
| **Setup Complexity** | Low | Medium | Medium | High |

### When to Choose Monzo Egress Operator

**Choose Monzo Egress Operator when:**
- ✅ Running on AWS EKS
- ✅ Want managed NAT Gateway (less operations)
- ✅ Need automatic provisioning via Kubernetes
- ✅ Prefer AWS native services
- ✅ Team has AWS expertise

**Consider alternatives when:**
- 📋 Multi-cloud deployment (choose Cilium/Antrea)
- 📋 Cost optimization critical (self-managed cheaper)
- 📋 Need >10 Gbps per gateway (use multiple NATs)
- 📋 On-premises deployment (choose Cilium/Antrea/Kube-OVN)

## Migration from Manual NAT Gateway

### Before: Manual NAT Gateway

```bash
# Manual AWS commands
aws ec2 create-nat-gateway --subnet-id subnet-xxx --allocation-id eipalloc-xxx
aws ec2 create-route --route-table-id rtb-xxx --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-xxx
```

### After: Egress Operator

```yaml
apiVersion: ec2.monzo.com/v1beta1
kind: Egress
metadata:
  name: production-egress
spec:
  region: us-east-1
  vpcId: vpc-12345678
  subnets:
    - subnet-xxx
```

### Migration Steps

```bash
# 1. Install egress operator
helm install egress-operator monzo/egress-operator ...

# 2. Create Egress CRD
kubectl apply -f egress.yaml

# 3. Verify NAT Gateway created
kubectl get egress production-egress

# 4. Update workloads to use new subnets
kubectl patch deployment app -p '{"spec":{"template":{"spec":{"nodeSelector":{"egress":"production"}}}}}'

# 5. Delete manual NAT Gateway (after validation)
aws ec2 delete-nat-gateway --nat-gateway-id nat-manual123
```

## Next Steps

In the next post of this series:

- **Cloud NAT Solutions** - GCP Cloud NAT, Azure Firewall
- Comparison with AWS NAT Gateway
- Multi-cloud egress strategies

## Conclusion

Monzo Egress Operator provides:

**Advantages:**
- ✅ Kubernetes-native CRD interface
- ✅ Automated NAT Gateway provisioning
- ✅ AWS managed service (less operations)
- ✅ Automatic Elastic IP management
- ✅ Route table automation
- ✅ Multi-AZ high availability
- ✅ Cost allocation via tags

**Considerations:**
- 📋 AWS only (not multi-cloud)
- 📋 NAT Gateway costs ($32/month minimum)
- 📋 10 Gbps limit per NAT Gateway
- 📋 Requires IAM permissions
- 📋 Vendor lock-in to AWS

For **AWS EKS clusters** wanting managed egress with Kubernetes-native operations, Monzo Egress Operator provides an excellent balance of automation and reliability.

