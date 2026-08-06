---
title: Kubernetes and Vault integration
url: https://devopstales.github.io/kubernetes/k8s-vault/
date: 2021-04-07
keywords: kubernetes deployment, kubernetes vs docker, Kubernetes Secuity, best Practices, rancher, mutating webhook, HashiCorp Vault
---


In this post I will show you how you can integrate HashiCorp Vault to Kubernetes easily thanks to [Bank-Vaults](https://banzaicloud.com/products/bank-vaults/) made by Banzaicloud.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

In a [previous post](https://devopstales.github.io/cloud/k8s-security/) I talked about how Kubernetes cluster store the Kubernetes Secrets in the etcd as base64 encoded text and not encrypted. This is the reason why using an external secret store should be a good idea.

### What is Bank-Vaults

Bank-Vaults provides various tools for Hashicorp Vault to make its use easier. It is a wrapper for the official Vault client with automatic token renewal, built in Kubernetes support, and a dynamic database credential provider. 

### Vhat is Hashicorp Vault

HashiCorp Vault is a secrets management solution that brokers access for both humans and machines, through programmatic access, to systems. Secrets can be stored, dynamically generated, and in the case of encryption, keys can be consumed as a service without the need to expose the underlying key materials.

![Example image](/img/include/vault01.webp)

### Install Bank-Vaults Operator

Ther is a Kubernetes Helm chart to deploy the Banzai Cloud Vault Operator. We will use this for deploy the HashiCorp Vault in HA mode with etcd as storage backend. As a dependency the chart installs an etcd operator that runs as root so we need to use [my predifinde PSP](https://devopstales.github.io/home/rke2-pod-security-policy/) to allow this.

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: vault
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: psp-rolebinding-vault
  namespace: vault
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system-unrestricted-psp-role
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: vault-operator
  namespace: vault
spec:
  repo: "https://kubernetes-charts.banzaicloud.com"
  chart: vault-operator
  targetNamespace: vault
  valuesContent: |-
    etcd-operator:
      enabled: "true"
      etcdOperator:
        commandArgs:
          cluster-wide: "true"
    psp:
      enabled: true
      vaultSA: "vault"
```

When the operator runs correctly we can deploy the CRD to create teh Vault cluster

```yaml
apiVersion: "vault.banzaicloud.com/v1alpha1"
kind: "Vault"
metadata:
  name: "vault"
spec:
  size: 2
  image: vault:1.6.2

  # Specify the ServiceAccount where the Vault Pod and the Bank-Vaults configurer/unsealer is running
  serviceAccount: vault

  # Specify how many nodes you would like to have in your etcd cluster
  # NOTE: -1 disables automatic etcd provisioning
  etcdSize: 1

  #resources:
  # vault:
  #    requests:
  #      memory: "256Mi"
  #      cpu: "100m"
  #    limits:
  #      memory: "512Mi"
  #      cpu: "250m"

  etcdPVCSpec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi

  etcdAnnotations:
    etcd.database.coreos.com/scope: clusterwide

  etcdVersion: "3.3.17"

  # Support for distributing the generated CA certificate Secret to other namespaces.
  # Define a list of namespaces or use ["*"] for all namespaces.
  caNamespaces:
    - "demo-app"
    - "default"

  # Describe where you would like to store the Vault unseal keys and root token.
  unsealConfig:
    kubernetes:
      secretNamespace: vault

  # A YAML representation of a final vault config file.
  # See https://www.vaultproject.io/docs/configuration/ for more information.
  config:
    storage:
      etcd:
        address: https://etcd-cluster:2379
        ha_enabled: "true"
        etcd_api: "v3"
    listener:
      tcp:
        address: "0.0.0.0:8200"
        tls_cert_file: /vault/tls/server.crt
        tls_key_file: /vault/tls/server.key
    api_addr: https://vault.vault:8200
    telemetry:
      statsd_address: localhost:9125
    ui: true

  externalConfig:
    policies:
      - name: allow_secrets
        rules: path "secret/*" {
                 capabilities = ["create", "read", "update", "delete", "list"]
               }

    # The auth block allows configuring Auth Methods in Vault.
    # See https://www.vaultproject.io/docs/auth/index.html for more information.
    auth:
      - type: kubernetes
        roles:
          # Allow every pod in the default namespace to use the secret kv store
          - name: default
            bound_service_account_names: default
            bound_service_account_namespaces: "*"
            policies: allow_secrets
            ttl: 1h

    secrets:
      - path: secret
        type: kv
        description: General secrets
        options:
          version: 2

```

### Deploy the mutating webhook

Banzaicloud created a mutating webhook to automate the injection of the secrets from Vault.

```yaml
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: vault-secrets-webhook
  namespace: vault
spec:
  repo: "https://kubernetes-charts.banzaicloud.com"
  chart: vault-secrets-webhook
  targetNamespace: vault
```

### Install vault cli and create secret

```bash
sudo yum install -y yum-utils
# OR
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo 
yum install -y vault
```

Configure the client to connect to the server:

```bash
export VAULT_TOKEN=$(kubectl get secrets vault-unseal-keys -o jsonpath={.data.vault-root} | base64 --decode)
kubectl get secret vault-tls -o jsonpath="{.data.ca\.crt}" | base64 --decode > $PWD/vault-ca.crt
export VAULT_CACERT=$PWD/vault-ca.crt
export VAULT_ADDR=https://127.0.0.1:8200
kubectl port-forward service/vault 8200 &

vault kv put secret/accounts/aws AWS_SECRET_ACCESS_KEY=s3cr3t
```

Now we start a container in the `demo-app` namespace and we us the `AWS_SECRET_ACCESS_KEY` variable from a secret stored in Vault.

```bash
nano 05_demo.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-secrets
  namespace: demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-secrets
  template:
    metadata:
      labels:
        app: hello-secrets
      annotations:
        vault.security.banzaicloud.io/vault-addr: "https://vault.vault:8200"
        vault.security.banzaicloud.io/vault-tls-secret: "vault-tls"
    spec:
      serviceAccountName: default
      containers:
      - name: nginx
        image: nginxinc/nginx-unprivileged
        command: ["sh", "-c", "echo $AWS_SECRET_ACCESS_KEY && echo going to sleep... && sleep 10000"]
        env:
        - name: AWS_SECRET_ACCESS_KEY
          value: "vault:secret/data/accounts/aws#AWS_SECRET_ACCESS_KEY"
```

```
kubectl apply -f 05_demo.yaml
kubectl logs hello-secrets-676b67c659-fvk9d -n demo-app
time="2021-04-05T08:45:11Z" level=info msg="received new Vault token" app=vault-env
time="2021-04-05T08:45:11Z" level=info msg="initial Vault token arrived" app=vault-env
time="2021-04-05T08:45:11Z" level=info msg="spawning process: [sh -c echo $AWS_SECRET_ACCESS_KEY && echo going to sleep... && sleep 10000]" app=vault-env
time="2021-04-05T08:45:11Z" level=info msg="renewed Vault token" app=vault-env ttl=1h0m0s
s3cr3t
going to sleep...
```

---
* https://banzaicloud.com/blog/kubernetes-oidc/

