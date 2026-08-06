---
title: Azure Key Vault AKS integration with CSI Driver
url: https://devopstales.github.io/cloud/aks-azure-key-vault-csi/
date: 2023-03-08
keywords: Azure, AKS, streaming
---



In this Post I will show you how you can use CSI Driver to mount secrets from Azure Key Vault to AKS.
<!--more-->

## Create key vault and add keys

First we need to create a key vault

```bash
az aks enable-addons \
--addons=azure-keyvault-secrets-provider \
--name=$CLUSTER \
--resource-group=$RG

# create the key vault and turn on Azure RBAC; we will grant a managed identity access to this key vault below
az keyvault create \
--name $KV \
--resource-group $RG \
--location westeurope \
--enable-rbac-authorization true

# get the subscription id
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
 
# get your user object id
USER_OBJECT_ID=$(az ad signed-in-user show --query objectId -o tsv)
 
# grant yourself access to key vault
az role assignment create \
--assignee-object-id $USER_OBJECT_ID \
--role "Key Vault Administrator" \
--scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG/providers/Microsoft.KeyVault/vaults/$KV
 
# add a secret to the key vault
az keyvault secret set --vault-name $KV --name $SECRET --value $VALUE
```

Create secret in keyvault:

```bash
az keyvault secret set \
--vault-name "$KV" \
--name "sqldatabase" \
--value 'kvdemo'

az keyvault secret set \
--vault-name "$KV" \
--name "sqlusername" \
--value 'root'

az keyvault secret set \
--vault-name "$KV" \
--name "sqlpassword" \
--value 'Password1'
```

## Acces key vault with system-assigned managed identity

If you created your AKS cluster with managed identity you can go to grant access to a managed identity:

```bash
# grab the managed identity principalId assuming it is in the default
# MC_ group for your cluster and resource group
IDENTITY_ID=$(az identity show -g MC\_$RG\_$CLUSTER\_westeurope --name azurekeyvaultsecretsprovider-$CLUSTER --query principalId -o tsv)
 
# grant access rights on Key Vault
az role assignment create \
--assignee-object-id $IDENTITY_ID \
--role "Key Vault Administrator" \
--scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG/providers/Microsoft.KeyVault/vaults/$KV
```

If not yet enabled you need to enable system-assigned managed identity on AKS:

```bash
az aks update \
--resource-group $RG \
--name  $CLUSTER \
--enable-managed-identity
```

Create a SecretProviderClass:

```bash
AZURE_TENANT_ID=$(az account show --query tenantId -o tsv)
CLIENT_ID=$(az aks show -g $RG -n $CLUSTER --query addonProfiles.azureKeyvaultSecretsProvider.identity.clientId -o tsv)
```

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: demo-secret
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    # use managed identity
    useVMManagedIdentity: "true"
    # add managed identity id manually
    userAssignedIdentityID: "$CLIENT_ID"
    tenantId: "$AZURE_TENANT_ID"
    keyvaultName: "$KV"
    # name and type in keyvault
    objects: |
      array:
        - |
          objectName: "sqldatabase"
          objectType: secret
        - |
          objectName: "sqlusername"
          objectType: secret
        - |
          objectName: "sqlpassword"
          objectType: secret
  secretObjects:
  - secretName: databasesecrets
    type: Opaque
    # name and key in secret
    data:
    - objectName: "sqldatabase"
      key: sqldatabase
    - objectName: "sqlusername"
      key: sqlusername
    - objectName: "sqlpassword"
      key: sqlpassword
EOF
```

Mount the secrets in pods

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: secretpods
  name: secretpods
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secretpods
  template:
    metadata:
      labels:
        app: secretpods
    spec:
      containers:
      - image: nginx
        name: nginx
        # get as environment variables
        env:
          - name:  sqldatabase
            valueFrom:
              secretKeyRef:
                name:  databasesecrets
                key:  sqldatabase
          - name:  sqlusername
            valueFrom:
              secretKeyRef:
                name:  databasesecrets
                key:  sqlusername
          - name:  sqlpassword
            valueFrom:
              secretKeyRef:
                name:  databasesecrets
                key:  sqlpassword
        # mount as file
        volumeMounts:
          - name:  secret-store
            mountPath:  "mnt/secret-store"
            readOnly: true
      # get the SecretProviderClass object
      volumes:
        - name:  secret-store
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "demo-secret"
EOF
```

## Acces key vault with service principal

Create an identity in Azure

```bash
export CLIENT_SECRET=$(az ad sp create-for-rbac \
--role Contributor --scopes /subscriptions/$SUBSCRIPTION_ID \
--name http://secrets-store-test --query 'password' -otsv)

export CLIENT_ID=$(az ad sp show --id http://secrets-store-test --query 'appId' -otsv)
```

Provide policy to the identity to access the Azure key vault

```bash
az keyvault set-policy -n $KV --secret-permissions get --spn $CLIENT_ID
```

Create the Kubernetes secret with credentials

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
data:
  clientid: $CLIENT_ID
  clientsecret: $CLIENT_SECRET
kind: Secret
metadata:
  labels:
    secrets-store.csi.k8s.io/used: "true"
  name: secrets-store-creds
  namespace: default
type: Opaque
EOF
```

Create and apply your own SecretProviderClass object

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: demo-secret
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    tenantId: "$AZURE_TENANT_ID"
    keyvaultName: "$KV"
    # name and type in keyvault
    objects: |
      array:
        - |
          objectName: "sqldatabase"
          objectType: secret
        - |
          objectName: "sqlusername"
          objectType: secret
        - |
          objectName: "sqlpassword"
          objectType: secret
  secretObjects:
  - secretName: databasesecrets
    type: Opaque
    # name and key in secret
    data:
    - objectName: "sqldatabase"
      key: sqldatabase
    - objectName: "sqlusername"
      key: sqlusername
    - objectName: "sqlpassword"
      key: sqlpassword
EOF
```

Mount the secrets in pods

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: secretpods
  name: secretpods
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secretpods
  template:
    metadata:
      labels:
        app: secretpods
    spec:
      containers:
      - image: nginx
        name: nginx
        # get as environment variables
        env:
          - name:  sqldatabase
            valueFrom:
              secretKeyRef:
                name:  databasesecrets
                key:  sqldatabase
          - name:  sqlusername
            valueFrom:
              secretKeyRef:
                name:  databasesecrets
                key:  sqlusername
          - name:  sqlpassword
            valueFrom:
              secretKeyRef:
                name:  databasesecrets
                key:  sqlpassword
        # mount as file
        volumeMounts:
          - name:  secret-store
            mountPath:  "mnt/secret-store"
            readOnly: true
      # get the SecretProviderClass object
      volumes:
        - name:  secret-store
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "demo-secret"
            # Only required when using service principal mode
            nodePublishSecretRef:
              name: secrets-store-creds
EOF
```

## Acces key vault with federated workload identity

This scanario is similar to the previous one, but instead of using managed identities we use the AKS cluster's workload identity to authenticate. In normal situation the identity is autenticates with a password. Password authentication is not the safest option. Instad we will use a federation to authenticate with OIDC SSO authentication.

Enable OIDC issuer and Workload identity in an existing AKS cluster.

```bash
az aks update \
--resource-group $RG \
--name  $CLUSTER \
--enable-oidc-issuer \
--enable-workload-identity
```

Create a managed identity:

```bash
az identity create \
--name secrets-store-test \
--resource-group $RG \
--location $RG_LOCATION \
--subscription $SUBSCRIPTION
```

Get the OIDC issuer url and export it as an environment variable.

```bash
export AKS_OIDC_ISSUER=$(az aks show --name $CLUSTER --resource-group $RG --query "oidcIssuerProfile.issuerUrl" -o tsv)
```

Export managed identity variable. We will assign this to the AKS service account.

```bash
export USER_ASSIGNED_CLIENT_ID=$(az identity show --resource-group $RG --name secrets-store-test --query 'clientId' -otsv)
export IDENTITY_TENANT=$(az aks show --name $CLUSTER --resource-group $RG)
```

Create AKS service account and we will assign the Managed Identity ClientID to it using `azure.workload.identity/client-id` annotation.

```yaml
az aks get-credentials -n $CLUSTER -g $RG

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  labels:
    azure.workload.identity/use: "true"
  name: keyvault-sa
  namespace: default
EOF
```

Create the federated identity credential between the managed identity, service account issuer and subject.

```bash
export FEDERATED_IDENTITY_NAME="aksfederatedidentity" # can be changed as needed
az identity federated-credential create \
--name $FEDERATED_IDENTITY_NAME \
--identity-name secrets-store-test \
--resource-group $RG \
--issuer ${AKS_OIDC_ISSUER} \
--subject system:serviceaccount:default:keyvault-sa
```

Provide policy to the identity to access the Azure key vault

```bash
az keyvault set-policy -n $KV --secret-permissions get --spn $USER_ASSIGNED_CLIENT_ID
```

Create a SecretProviderClass:

```bash
AZURE_TENANT_ID=$(az account show --query tenantId -o tsv)
```

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: demo-secret
  namespace: default
spec:
  provider: azure
  parameters:
    # use managed identity
    useVMManagedIdentity: "false"
    clientID: "$USER_ASSIGNED_CLIENT_ID"
    tenantId: "$AZURE_TENANT_ID"
    keyvaultName: "$KV"
    # name and type in keyvault
    objects: |
      array:
        - |
          objectName: "sqldatabase"
          objectType: secret
        - |
          objectName: "sqlusername"
          objectType: secret
        - |
          objectName: "sqlpassword"
          objectType: secret
  secretObjects:
  - secretName: databasesecrets
    type: Opaque
    # name and key in secret
    data:
    - objectName: "sqldatabase"
      key: sqldatabase
    - objectName: "sqlusername"
      key: sqlusername
    - objectName: "sqlpassword"
      key: sqlpassword
EOF
```

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: secretpods
  name: secretpods
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secretpods
  template:
    metadata:
      labels:
        app: secretpods
    spec:
      serviceAccountName: keyvault-sa
      containers:
      - image: nginx
        name: nginx
        # get as environment variables
        env:
          - name:  sqldatabase
            valueFrom:
              secretKeyRef:
                name:  databasesecrets
                key:  sqldatabase
          - name:  sqlusername
            valueFrom:
              secretKeyRef:
                name:  databasesecrets
                key:  sqlusername
          - name:  sqlpassword
            valueFrom:
              secretKeyRef:
                name:  databasesecrets
                key:  sqlpassword
        # mount as file
        volumeMounts:
          - name:  secret-store
            mountPath:  "mnt/secret-store"
            readOnly: true
      # get the SecretProviderClass object
      volumes:
        - name:  secret-store
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "demo-secret"
EOF
```

## Demo time

Now we you can werify the secret in the pod:

```bash
export POD_NAME=$(kubectl get pods -n default -l "app=secretpods" -o jsonpath="{.items[0].metadata.name}")
 
# if this does not work, check the status of the pod
# if still in ContainerCreating there might be an issue
kubectl exec -it $POD_NAME -- sh
 
cd /mnt/secret-store
ls
sqldatabase  sqlpassword  sqlusername
# the file containing the secret is listed
cat sqldatabase
 
# echo the value of the environment variable
echo $sqldatabase
```

