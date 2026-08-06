---
title: Create EKS Cluster with eksctl
url: https://devopstales.github.io/cloud/aws-eks-install/
date: 2022-02-11
keywords: kubernetes deployment, CNI, CNI, Container Network Interface, calico, AWS, EKS, Elastic Kubernetes Registry, VPC CNI, eksctl
---


In this pos I will show you how you can install an AWS managed Elastic Kubernetes Service with ekscli.
<!--more-->

### Create an AWS KMS Custom Managed Key

Create a CMK for the EKS cluster to use when encrypting your Kubernetes secrets:

```bash
aws kms create-alias --alias-name alias/devopstales --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
```

### Create EKS Cluster with eksctl
`eksctl` is a simple CLI tool for creating clusters on EKS. To download the latest release, run:

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

```bash
# Let’s start by setting a few environment variables:
export AWS_REGION=eu-central-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
export EKS_CLUSTER_NAME=eks-devopstales

# Get the KMS Custom Managed Key resource name:
export MASTER_ARN=$(aws kms describe-key --key-id alias/devopstales --query KeyMetadata.Arn --output text)

# Create a cluster
eksctl create cluster \
  --name $EKS_CLUSTER_NAME \
  --region $AWS_REGION \
  --managed
```

If ylou need mor advanced configuration You can create a deployment file for `eksctl`:

```yaml
cat << EOF > eks_devopstales.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: eks-devopstales
  region: ${AWS_REGION}
  version: "1.21"
availabilityZones: ["eu-central-1a"]
managedNodeGroups:
- name: workers
  labels: 
      role: workers
      environment: poc
  desiredCapacity: 1
  instanceType: t3.small
  volumeEncrypted: true
  volumeSize: 30
  ssh:
    enableSsm: true
secretsEncryption:
  keyARN: ${MASTER_ARN}
# To enable all of the control plane logs, uncomment below:
# cloudWatch:
#  clusterLogging:
#    enableTypes: ["*"]
EOF
```

Next, use the file you created as the input for the eksctl cluster creation:

```bash
eksctl create cluster -f eks_devopstales.yaml
```

Test The cluster

```bash
kubectl get nodes
```

### Using IAM Groups to manage Kubernetes cluster access

We are going to create 3 roles:
* **k8sAdmin** role which will have admin rights in our EKS cluster
* **k8sDev** role which will give access to the developers namespace in our EKS cluster
* **k8sInteg** role which will give access to the integration namespace in our EKS cluster

Create the roles:

```bash
POLICY=$(echo -n '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':root"},"Action":"sts:AssumeRole","Condition":{}}]}')

echo ACCOUNT_ID=$ACCOUNT_ID
echo POLICY=$POLICY

aws iam create-role \
  --role-name k8sAdmin \
  --description "Kubernetes administrator role (for AWS IAM Authenticator for Kubernetes)." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'

aws iam create-role \
  --role-name k8sDev \
  --description "Kubernetes developer role (for AWS IAM Authenticator for Kubernetes)." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'
  
aws iam create-role \
  --role-name k8sInteg \
  --description "Kubernetes role for integration namespace in quick cluster." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'

```

We will define 3 groups:
* **k8sAdmin** - users from this group will have admin rights on the kubernetes cluster
* **k8sDe** - users from this group will have full access only in the development namespace of the cluster
* **k8sInteg** - users from this group will have access to integration namespace.

Create k8sAdmin IAM Group:

```bash
aws iam create-group --group-name k8sAdmin

ADMIN_GROUP_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeOrganizationAccountRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':role/k8sAdmin"
    }
  ]
}')
echo ADMIN_GROUP_POLICY=$ADMIN_GROUP_POLICY

aws iam put-group-policy \
--group-name k8sAdmin \
--policy-name k8sAdmin-policy \
--policy-document "$ADMIN_GROUP_POLICY"
```

Create k8sDev IAM Group:

```bash
aws iam create-group --group-name k8sDev

DEV_GROUP_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeOrganizationAccountRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':role/k8sDev"
    }
  ]
}')
echo DEV_GROUP_POLICY=$DEV_GROUP_POLICY

aws iam put-group-policy \
--group-name k8sDev \
--policy-name k8sDev-policy \
--policy-document "$DEV_GROUP_POLICY"
```

Create k8sInteg IAM Group:

```bash
aws iam create-group --group-name k8sInteg

INTEG_GROUP_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeOrganizationAccountRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':role/k8sInteg"
    }
  ]
}')
echo INTEG_GROUP_POLICY=$INTEG_GROUP_POLICY

aws iam put-group-policy \
--group-name k8sInteg \
--policy-name k8sInteg-policy \
--policy-document "$INTEG_GROUP_POLICY"
```

Create IAM Users and add to the groups:

```bash
aws iam create-user --user-name PaulAdmin
aws iam create-user --user-name JeanDev
aws iam create-user --user-name PierreInteg

aws iam add-user-to-group --group-name k8sAdmin --user-name PaulAdmin
aws iam add-user-to-group --group-name k8sDev --user-name JeanDev
aws iam add-user-to-group --group-name k8sInteg --user-name PierreInteg

# Get Access key for the users
aws iam create-access-key --user-name PaulAdmin | tee /tmp/PaulAdmin.json
aws iam create-access-key --user-name JeanDev | tee /tmp/JeanDev.json
aws iam create-access-key --user-name PierreInteg | tee /tmp/PierreInteg.json
```

### Configure Kubernetes RBAC 

```bash
kubectl create namespace integration
kubectl create namespace development
```

Configuring access to development namespace:

```yaml
cat << EOF | kubectl apply -f - -n development
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dev-role
rules:
  - apiGroups:
      - ""
      - "apps"
      - "batch"
      - "extensions"
    resources:
      - "configmaps"
      - "cronjobs"
      - "deployments"
      - "events"
      - "ingresses"
      - "jobs"
      - "pods"
      - "pods/attach"
      - "pods/exec"
      - "pods/log"
      - "pods/portforward"
      - "secrets"
      - "services"
    verbs:
      - "create"
      - "delete"
      - "describe"
      - "get"
      - "list"
      - "patch"
      - "update"
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dev-role-binding
subjects:
- kind: User
  name: dev-user
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

Configuring access to integration namespace:

```yaml
cat << EOF | kubectl apply -f - -n integration
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: integ-role
rules:
  - apiGroups:
      - ""
      - "apps"
      - "batch"
      - "extensions"
    resources:
      - "configmaps"
      - "cronjobs"
      - "deployments"
      - "events"
      - "ingresses"
      - "jobs"
      - "pods"
      - "pods/attach"
      - "pods/exec"
      - "pods/log"
      - "pods/portforward"
      - "secrets"
      - "services"
    verbs:
      - "create"
      - "delete"
      - "describe"
      - "get"
      - "list"
      - "patch"
      - "update"
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: integ-role-binding
subjects:
- kind: User
  name: integ-user
roleRef:
  kind: Role
  name: integ-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

Gives Access to our IAM Roles to EKS Cluster:

```bash
eksctl create iamidentitymapping \
  --cluster eksworkshop-eksctl \
  --arn arn:aws:iam::${ACCOUNT_ID}:role/k8sDev \
  --username dev-user

eksctl create iamidentitymapping \
  --cluster eksworkshop-eksctl \
  --arn arn:aws:iam::${ACCOUNT_ID}:role/k8sInteg \
  --username integ-user

eksctl create iamidentitymapping \
  --cluster eksworkshop-eksctl \
  --arn arn:aws:iam::${ACCOUNT_ID}:role/k8sAdmin \
  --username admin \
  --group system:masters
```

