---
title: Kubernetes Egress Gateway with Cloud NAT Solutions
url: https://devopstales.github.io/kubernetes/cloud-nat-egress-solutions/
date: 2026-04-18
keywords: Cloud NAT, AWS NAT Gateway, GCP Cloud NAT, Azure NAT Gateway, Azure Firewall, Kubernetes egress, managed egress, cloud egress
---


Cloud providers offer managed NAT services that provide egress control for Kubernetes clusters without the operational overhead of self-managed solutions. This post covers native cloud NAT options from AWS, GCP, and Azure with practical configuration examples.

<!--more-->

{{< content "/filedir/egress-gateway-series.html" >}}

## Why Cloud NAT?

Managed cloud NAT services provide:

- ✅ **Zero operations** - Fully managed by cloud provider
- ✅ **High availability** - Built-in redundancy across zones
- ✅ **Auto-scaling** - Handles traffic spikes automatically
- ✅ **No maintenance** - Provider handles updates and patches
- ✅ **Cloud integration** - Native integration with cloud services
- ✅ **Predictable pricing** - Pay-per-use or fixed pricing

### Trade-offs

| Advantage | Consideration |
|-----------|---------------|
| Managed service | Vendor lock-in |
| Auto-scaling | Less control over configuration |
| High availability | Cloud-specific (not portable) |
| No operations | Higher cost than self-managed |

## AWS NAT Gateway

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      AWS VPC                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 Private Subnet                      │    │
│  │                                                     │    │
│  │   ┌──────────┐                                      │    │
│  │   │ EKS Pod  │──┐                                   │    │
│  │   └──────────┘  │                                   │    │
│  │                 ▼                                   │    │
│  │   ┌──────────┐                                      │    │
│  │   │ EKS Pod  │──┼──> ┌─────────┐                   │    │
│  │   └──────────┘  │    │ EKS     │                   │    │
│  │                 │    │ Node    │                   │    │
│  │                 │    └────┬────┘                   │    │
│  │                 │         │                         │    │
│  │                 └─────────┼──────────────────────┐  │    │
│  │                           ▼                      │  │    │
│  │                    ┌──────────────┐              │  │    │
│  │                    │ NAT Gateway  │──────────────┼──┼───>│
│  │                    │              │              │  │    │
│  │                    └──────────────┘              │  │    │
│  │                           │                      │  │    │
│  │                           ▼                      │  │    │
│  │                    ┌──────────────┐              │  │    │
│  │                    │Internet Gw   │──────────────┼──┼───>│
│  │                    │              │              │  │    │
│  │                    └──────────────┘              │  │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │  External APIs   │
                    └──────────────────┘
```

### Key Features

| Feature | Description |
|---------|-------------|
| **Availability** | Multi-AZ deployment for HA |
| **Bandwidth** | Up to 100 Gbps |
| **Scaling** | Automatic up to capacity limit |
| **Pricing** | Per hour + per GB processed |
| **Integration** | VPC route tables, Security Groups |

### Pricing (us-east-1)

| Component | Cost |
|-----------|------|
| **NAT Gateway-hour** | $0.045 per hour |
| **Data processed** | $0.045 per GB |
| **Multi-AZ** | 1 NAT Gateway per AZ |

**Estimated Monthly Cost:**
- Small cluster (< 100 GB/day): ~$50-100/month
- Medium cluster (100-500 GB/day): ~$200-500/month
- Large cluster (> 500 GB/day): ~$500-2000/month

### Configuration

#### Step 1: Create NAT Gateway

```bash
# Allocate Elastic IP
EIP_ALLOC=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text)

# Create NAT Gateway in public subnet
NAT_GW=$(aws ec2 create-nat-gateway \
  --subnet-id subnet-public-123 \
  --allocation-id $EIP_ALLOC \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=eks-nat-gw}]' \
  --query 'NatGatewayId' \
  --output text)

# Wait for NAT Gateway to be available
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW

echo "NAT Gateway created: $NAT_GW"
```

#### Step 2: Update Route Tables

```bash
# Get private subnet route table
ROUTE_TABLE=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=eks-private-subnet" \
  --query 'RouteTables[0].RouteTableId' \
  --output text)

# Add default route to NAT Gateway
aws ec2 create-route \
  --route-table-id $ROUTE_TABLE \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW
```

#### Step 3: Verify Configuration

```bash
# Check NAT Gateway status
aws ec2 describe-nat-gateways \
  --nat-gateway-ids $NAT_GW \
  --query 'NatGateways[0].{State:State,Ip:NatGatewayAddresses[0].PublicIp}'

# Check route table
aws ec2 describe-route-tables \
  --route-table-ids $ROUTE_TABLE \
  --query 'RouteTables[0].Routes[?DestinationCidrBlock==`0.0.0.0/0`]'
```

### Multi-AZ High Availability

```bash
# Create NAT Gateway in each AZ
for AZ in us-east-1a us-east-1b us-east-1c; do
  # Get public subnet for AZ
  SUBNET=$(aws ec2 describe-subnets \
    --filters "Name=tag:Name,Values=eks-public-$AZ" \
    --query 'Subnets[0].SubnetId' \
    --output text)
  
  # Allocate EIP
  EIP=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
  
  # Create NAT Gateway
  aws ec2 create-nat-gateway \
    --subnet-id $SUBNET \
    --allocation-id $EIP \
    --tag-specifications "ResourceType=natgateway,Tags=[{Key=Name,Value=eks-nat-$AZ}]"
done
```

### EKS-Specific Configuration

```yaml
# eksctl cluster config with NAT Gateway
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
        id: subnet-priv-1a
        natGatewayId: nat-0abc123
      us-east-1b:
        id: subnet-priv-1b
        natGatewayId: nat-0def456
    public:
      us-east-1a:
        id: subnet-pub-1a
      us-east-1b:
        id: subnet-pub-1b

# NAT Gateways are created automatically by eksctl
# One per AZ with private subnets
```

### Monitoring with CloudWatch

```bash
# Create CloudWatch alarm for high traffic
aws cloudwatch put-metric-alarm \
  --alarm-name "NAT-HighBytesOut" \
  --metric-name BytesOutToDestination \
  --namespace AWS/NatGateway \
  --statistic Sum \
  --period 300 \
  --threshold 1000000000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT:alerts \
  --dimensions Name=NatGatewayId,Value=nat-0abc123

# View NAT Gateway metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/NatGateway \
  --metric-name BytesOutToDestination \
  --dimensions Name=NatGatewayId,Value=nat-0abc123 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Sum
```

### Cost Optimization

```bash
# Enable VPC Flow Logs to analyze traffic
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-12345678 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flow-logs \
  --iam-role-arn arn:aws:iam::ACCOUNT:role/flow-logs-role

# Analyze traffic patterns to optimize
aws logs filter-log-events \
  --log-group-name /aws/vpc/flow-logs \
  --filter-pattern "dstAddr != 10.0.0.0/8" \
  --start-time $(date -u -d '7 days ago' +%s)000 \
  --end-time $(date -u +%s)000
```

## GCP Cloud NAT

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      GCP VPC                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 Private Subnet                      │    │
│  │                                                     │    │
│  │   ┌──────────┐                                      │    │
│  │   │ GKE Pod  │──┐                                   │    │
│  │   └──────────┘  │                                   │    │
│  │                 ▼                                   │    │
│  │   ┌──────────┐                                      │    │
│  │   │ GKE Pod  │──┼──> ┌─────────┐                   │    │
│  │   └──────────┘  │    │ GKE     │                   │    │
│  │                 │    │ Node    │                   │    │
│  │                 │    └────┬────┘                   │    │
│  │                 │         │                         │    │
│  │                 └─────────┼──────────────────────┐  │    │
│  │                           ▼                      │  │    │
│  │                    ┌──────────────┐              │  │    │
│  │                    │  Cloud NAT   │──────────────┼──┼───>│
│  │                    │              │              │  │    │
│  │                    └──────────────┘              │  │    │
│  │                           │                      │  │    │
│  │                           ▼                      │  │    │
│  │                    ┌──────────────┐              │  │    │
│  │                    │Cloud Router  │──────────────┼──┼───>│
│  │                    │              │              │  │    │
│  │                    └──────────────┘              │  │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │    Internet      │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  External APIs   │
                    └──────────────────┘
```

### Key Features

| Feature | Description |
|---------|-------------|
| **Availability** | Regional service (multi-zone) |
| **Bandwidth** | Auto-scales up to 100+ Gbps |
| **Scaling** | Automatic based on traffic |
| **Pricing** | Per VM + per GB processed |
| **Integration** | Cloud Router, VPC, Firewall Rules |

### Pricing (us-central1)

| Component | Cost |
|-----------|------|
| **Cloud NAT usage** | $0.008 per VM per hour |
| **Data processing** | $0.01 per GB (first 1TB) |
| **Cloud Router** | $0.01 per hour |

**Estimated Monthly Cost:**
- Small cluster (10 nodes, < 100 GB): ~$100-200/month
- Medium cluster (50 nodes, 100-500 GB): ~$500-1000/month
- Large cluster (100+ nodes, > 500 GB): ~$1000-3000/month

### Configuration

#### Step 1: Create Cloud Router

```bash
# Create Cloud Router in region
gcloud compute routers create gke-nat-router \
  --region us-central1 \
  --network default \
  --asn 64514

# Verify router
gcloud compute routers describe gke-nat-router \
  --region us-central1
```

#### Step 2: Create Cloud NAT

```bash
# Create Cloud NAT configuration
gcloud compute routers nats create gke-nat-config \
  --router=gke-nat-router \
  --region=us-central1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges \
  --enable-logging

# Verify NAT configuration
gcloud compute routers nats describe gke-nat-config \
  --router=gke-nat-router \
  --region=us-central1
```

#### Step 3: Configure for GKE

```bash
# Get GKE cluster details
gcloud container clusters describe my-cluster \
  --region us-central1 \
  --format="value(network)"

# Ensure private cluster configuration
gcloud container clusters update my-cluster \
  --region us-central1 \
  --enable-master-authorized-networks \
  --master-authorized-networks 10.0.0.0/8
```

### Advanced Configuration

#### Custom NAT IP Addresses

```bash
# Reserve static external IPs
gcloud compute addresses create nat-ip-1 \
  --region=us-central1

gcloud compute addresses create nat-ip-2 \
  --region=us-central1

# Create NAT with specific IPs
gcloud compute routers nats create gke-nat-custom-ip \
  --router=gke-nat-router \
  --region=us-central1 \
  --nat-external-ip-pool=nat-ip-1,nat-ip-2 \
  --nat-all-subnet-ip-ranges \
  --enable-logging
```

#### Port Allocation Tuning

```bash
# Configure port allocation for high-traffic workloads
gcloud compute routers nats create gke-nat-optimized \
  --router=gke-nat-router \
  --region=us-central1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges \
  --min-ports-per-vm=1024 \
  --max-ports-per-vm=65536 \
  --tcp-time-wait-timeout=120 \
  --udp-idle-timeout=60 \
  --icmp-idle-timeout=60 \
  --log-config=ENABLED \
  --log-filter=ERRORS_ONLY
```

### Monitoring with Cloud Monitoring

```bash
# Create monitoring dashboard
gcloud monitoring dashboards create --config='
{
  "displayName": "Cloud NAT Dashboard",
  "gridLayout": {
    "widgets": [
      {
        "title": "NAT Gateway Bytes",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": {
              "timeSeriesFilter": {
                "filter": "metric.type=\"compute.googleapis.com/nat/bytes_sent\""
              }
            }
          }]
        }
      },
      {
        "title": "NAT Gateway Packets",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": {
              "timeSeriesFilter": {
                "filter": "metric.type=\"compute.googleapis.com/nat/packets_sent\""
              }
            }
          }]
        }
      }
    ]
  }
}'

# Set up alerting for high traffic
gcloud alpha monitoring policies create --config='
{
  "displayName": "Cloud NAT High Egress",
  "conditions": [
    {
      "displayName": "Egress bytes > 100GB/hour",
      "conditionThreshold": {
        "filter": "metric.type=\"compute.googleapis.com/nat/bytes_sent\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 100000000000,
        "duration": "300s"
      }
    }
  ],
  "notificationChannels": ["projects/PROJECT/notificationChannels/CHANNEL_ID"]
}'
```

### Cost Optimization

```bash
# Analyze NAT usage by subnet
gcloud compute routers nats get-iam-policy gke-nat-config \
  --router=gke-nat-router \
  --region=us-central1

# Enable detailed logging for analysis
gcloud compute routers nats update gke-nat-config \
  --router=gke-nat-router \
  --region=us-central1 \
  --log-config=ENABLED \
  --log-filter=ALL

# Review logs in Cloud Logging
gcloud logging read "resource.type=gce_nat" \
  --limit=50 \
  --format="table(timestamp,jsonPayload)"
```

## Azure NAT Gateway

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Azure VNet                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  AKS Subnet                         │    │
│  │                                                     │    │
│  │   ┌──────────┐                                      │    │
│  │   │ AKS Pod  │──┐                                   │    │
│  │   └──────────┘  │                                   │    │
│  │                 ▼                                   │    │
│  │   ┌──────────┐                                      │    │
│  │   │ AKS Pod  │──┼──> ┌─────────┐                   │    │
│  │   └──────────┘  │    │ AKS     │                   │    │
│  │                 │    │ Node    │                   │    │
│  │                 │    └────┬────┘                   │    │
│  │                 │         │                         │    │
│  │                 └─────────┼──────────────────────┐  │    │
│  │                           ▼                      │  │    │
│  │                    ┌──────────────┐              │  │    │
│  │                    │  NAT Gateway │──────────────┼──┼───>│
│  │                    │              │              │  │    │
│  │                    └──────────────┘              │  │    │
│  │                           │                      │  │    │
│  │                           ▼                      │  │    │
│  │                    ┌──────────────┐              │  │    │
│  │                    │  Public IP   │──────────────┼──┼───>│
│  │                    │              │              │  │    │
│  │                    └──────────────┘              │  │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │    Internet      │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  External APIs   │
                    └──────────────────┘
```

### Key Features

| Feature | Description |
|---------|-------------|
| **Availability** | Zone-redundant option |
| **Bandwidth** | Up to 50 Gbps per instance |
| **Scaling** | Manual (add more IPs) |
| **Pricing** | Per hour + per GB processed |
| **Integration** | VNet, NSG, Azure Firewall |

### Pricing (East US)

| Component | Cost |
|-----------|------|
| **NAT Gateway** | $0.045 per hour |
| **Data processed** | $0.045 per GB |
| **Public IP** | $0.005 per IP per hour |

**Estimated Monthly Cost:**
- Small cluster (< 100 GB): ~$50-150/month
- Medium cluster (100-500 GB): ~$200-600/month
- Large cluster (> 500 GB): ~$600-2000/month

### Configuration

#### Step 1: Create Public IP

```bash
# Create public IP for NAT Gateway
az network public-ip create \
  --resource-group my-aks-rg \
  --name nat-gateway-ip \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3

# Get public IP ID
PUBLIC_IP_ID=$(az network public-ip show \
  --resource-group my-aks-rg \
  --name nat-gateway-ip \
  --query id \
  --output tsv)
```

#### Step 2: Create NAT Gateway

```bash
# Create NAT Gateway
az network nat gateway create \
  --resource-group my-aks-rg \
  --name aks-nat-gateway \
  --public-ip-addresses $PUBLIC_IP_ID \
  --idle-timeout 4

# Verify NAT Gateway
az network nat gateway show \
  --resource-group my-aks-rg \
  --name aks-nat-gateway
```

#### Step 3: Associate with AKS Subnet

```bash
# Get AKS subnet ID
SUBNET_ID=$(az network vnet subnet show \
  --resource-group my-aks-rg \
  --vnet-name aks-vnet \
  --name aks-subnet \
  --query id \
  --output tsv)

# Associate NAT Gateway with subnet
az network vnet subnet update \
  --resource-group my-aks-rg \
  --vnet-name aks-vnet \
  --name aks-subnet \
  --nat-gateway aks-nat-gateway

# Verify association
az network vnet subnet show \
  --resource-group my-aks-rg \
  --vnet-name aks-vnet \
  --name aks-subnet \
  --query natGateway
```

### Azure Firewall Alternative

For advanced egress control with L7 filtering:

```bash
# Create Azure Firewall
az network firewall create \
  --resource-group my-aks-rg \
  --name aks-firewall \
  --sku AZFW_Hub

# Create public IP for Firewall
az network public-ip create \
  --resource-group my-aks-rg \
  --name firewall-ip \
  --sku Standard

# Associate IP with Firewall
az network firewall update \
  --name aks-firewall \
  --resource-group my-aks-rg \
  --public-ip firewall-ip

# Create firewall policy
az network firewall policy create \
  --resource-group my-aks-rg \
  --name aks-firewall-policy

# Add application rules (L7 filtering)
az network firewall policy rule-collection-group collection add-filter \
  --resource-group my-aks-rg \
  --policy-name aks-firewall-policy \
  --collection-group-name DefaultCollectionGroup \
  --collection-name AppRules \
  --priority 100 \
  --rule-type ApplicationRule \
  --action Allow \
  --rules web-access \
  --protocols Https=443 Http=80 \
  --source-addresses '*' \
  --target-fqdns '*.api.github.com' '*.docker.io' 'packages.microsoft.com'

# Associate policy with firewall
az network firewall update \
  --name aks-firewall \
  --resource-group my-aks-rg \
  --firewall-policy aks-firewall-policy
```

### AKS Integration

```bash
# Create AKS cluster with NAT Gateway
az aks create \
  --resource-group my-aks-rg \
  --name my-aks-cluster \
  --node-count 3 \
  --network-plugin azure \
  --vnet-subnet-id $SUBNET_ID \
  --enable-managed-identity

# For existing cluster, update subnet association
az network vnet subnet update \
  --resource-group my-aks-rg \
  --vnet-name aks-vnet \
  --name aks-subnet \
  --nat-gateway aks-nat-gateway
```

### Monitoring with Azure Monitor

```bash
# Enable diagnostic logs
az network nat gateway show \
  --resource-group my-aks-rg \
  --name aks-nat-gateway \
  --query id \
  --output tsv | xargs -I {} az monitor diagnostic-settings create \
    --name nat-gateway-logs \
    --resource {} \
    --workspace /subscriptions/SUB_ID/resourcegroups/DefaultResourceGroup-EUS/providers/microsoft.operationalinsights/workspaces/DefaultWorkspace-SUB_ID-EUS \
    --logs '[{"category": "AllMetrics", "enabled": true}]'

# Create alert for high egress
az monitor metrics alert create \
  --name high-nat-egress \
  --resource-group my-aks-rg \
  --scopes $(az network nat gateway show --resource-group my-aks-rg --name aks-nat-gateway --query id --output tsv) \
  --condition "total BytesOut > 100000000000" \
  --evaluation-interval 5m \
  --window-size 5m \
  --action /subscriptions/SUB_ID/resourceGroups/my-aks-rg/providers/microsoft.insights/actionGroups/alert-action-group
```

### Cost Optimization

```bash
# Analyze NAT Gateway costs
az consumption usage list \
  --start-date $(date -d '30 days ago' +%Y-%m-%d) \
  --end-date $(date +%Y-%m-%d) \
  --query "[?contains(meterDetails,'NAT Gateway')]" \
  --output table

# Review traffic patterns
az monitor metrics list \
  --resource $(az network nat gateway show --resource-group my-aks-rg --name aks-nat-gateway --query id --output tsv) \
  --metric "BytesOut" \
  --interval PT1H \
  --start-time $(date -d '7 days ago' -u +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)
```

## Comparison: Cloud NAT Solutions

| Feature | AWS NAT Gateway | GCP Cloud NAT | Azure NAT Gateway |
|---------|----------------|---------------|-------------------|
| **Availability** | Per-AZ | Regional | Zone-redundant |
| **Max Bandwidth** | 100 Gbps | 100+ Gbps | 50 Gbps |
| **Auto-scaling** | ✅ Yes | ✅ Yes | ⚠️ Manual (add IPs) |
| **Pricing Model** | Hour + GB | VM + GB | Hour + GB |
| **Static IP** | ✅ Elastic IP | ✅ Reserved IP | ✅ Public IP |
| **Logging** | ✅ VPC Flow Logs | ✅ Cloud Logging | ✅ Diagnostic Logs |
| **Multi-AZ** | Manual (per AZ) | ✅ Automatic | ✅ Zone-redundant |
| **Integration** | VPC, Route Tables | Cloud Router, VPC | VNet, NSG, Firewall |

### Cost Comparison (Monthly Estimate)

| Cluster Size | AWS | GCP | Azure |
|--------------|-----|-----|-------|
| **Small** (10 nodes, 50GB/day) | ~$70 | ~$120 | ~$75 |
| **Medium** (50 nodes, 200GB/day) | ~$350 | ~$600 | ~$380 |
| **Large** (100 nodes, 500GB/day) | ~$900 | ~$1500 | ~$950 |

*Note: Prices vary by region and actual traffic patterns*

## When to Choose Cloud NAT

**Choose Cloud NAT when:**
- ✅ Running on single cloud provider
- ✅ Want zero operational overhead
- ✅ Need high availability out-of-box
- ✅ Budget allows for managed service premium
- ✅ Cloud-native architecture preferred

**Consider self-managed when:**
- 📋 Multi-cloud deployment required
- 📋 Cost optimization is critical
- 📋 Need fine-grained control over NAT behavior
- 📋 Require custom routing policies
- 📋 Team has networking expertise

## Best Practices

### 1. Multi-AZ/Zone Deployment

```bash
# AWS: One NAT Gateway per AZ
for az in us-east-1a us-east-1b us-east-1c; do
  aws ec2 create-nat-gateway \
    --subnet-id subnet-public-$az \
    --allocation-id $(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
done

# Azure: Zone-redundant NAT Gateway
az network nat gateway create \
  --resource-group my-rg \
  --name nat-gateway \
  --public-ip-addresses $IP_IDS \
  --zone 1 2 3
```

### 2. Monitoring and Alerting

```bash
# Set up alerts for all cloud providers
# AWS
aws cloudwatch put-metric-alarm \
  --alarm-name NAT-HighTraffic \
  --metric-name BytesOutToDestination \
  --namespace AWS/NatGateway \
  --threshold 100000000000 \
  --comparison-operator GreaterThanThreshold

# GCP
gcloud alpha monitoring policies create --config='
{
  "displayName": "Cloud NAT High Egress",
  "conditions": [{
    "conditionThreshold": {
      "filter": "metric.type=\"compute.googleapis.com/nat/bytes_sent\"",
      "comparison": "COMPARISON_GT",
      "thresholdValue": 100000000000
    }
  }]
}'

# Azure
az monitor metrics alert create \
  --name high-nat-egress \
  --condition "total BytesOut > 100000000000"
```

### 3. Cost Optimization

- **Right-size NAT capacity** - Monitor and adjust based on actual usage
- **Use compression** - Reduce data transfer costs
- **Cache external content** - Reduce repeated egress traffic
- **Review traffic patterns** - Identify and optimize high-traffic destinations
- **Consider reserved pricing** - Some clouds offer committed use discounts

### 4. Security

```bash
# AWS: Restrict NAT Gateway access with Security Groups
aws ec2 create-security-group \
  --group-name nat-sg \
  --description "Security group for NAT Gateway"

# GCP: Use firewall rules to control egress
gcloud compute firewall-rules create allow-egress \
  --direction EGRESS \
  --action ALLOW \
  --rules tcp:443,tcp:80 \
  --destination-ranges 0.0.0.0/0

# Azure: Use NSG rules
az network nsg rule create \
  --resource-group my-rg \
  --nsg-name aks-nsg \
  --name AllowEgressHTTPS \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 443
```

## Next Steps

In the final post of this series:

- **Comprehensive comparison** of all 7 egress gateway solutions
- **Decision matrix** based on use cases
- **Cost analysis** across all solutions
- **Migration strategies** between solutions

## Conclusion

Cloud NAT solutions provide:

**Advantages:**
- ✅ Zero operational overhead (fully managed)
- ✅ Built-in high availability
- ✅ Automatic scaling
- ✅ Native cloud integration
- ✅ Predictable pricing
- ✅ No maintenance required

**Considerations:**
- 📋 Vendor lock-in to cloud provider
- 📋 Higher cost than self-managed
- 📋 Less control over configuration
- 📋 Cloud-specific (not portable)
- 📋 Limited customization options

For organizations running **single-cloud Kubernetes deployments** wanting to minimize operational complexity, cloud-native NAT services provide an excellent balance of reliability and simplicity.

---

*This is Part 8 of our 9-part series on Kubernetes egress gateway solutions. Continue to [Part 9: Comparison & Recommendations](/kubernetes/egress-gateway-comparison-guide/) for a comprehensive comparison of all solutions with decision matrices and final recommendations.*

