---
title: Kubernetes integration with external Vault
url: https://devopstales.github.io/kubernetes/k8s-vault-v2/
date: 2021-06-05
keywords: kubernetes deployment, kubernetes vs docker, Kubernetes Secuity, best Practices, rancher, mutating webhook, HashiCorp Vault, k3s
---


In this post I will show you how you can integrate an external HashiCorp Vault to Kubernetes.

<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

### Vhat is Hashicorp Vault

HashiCorp Vault is a secrets management solution that brokers access for both humans and machines, through programmatic access, to systems. Secrets can be stored, dynamically generated, and in the case of encryption, keys can be consumed as a service without the need to expose the underlying key materials.

![Example image](/img/include/vault-k8s-auth-workflow.webp)

### K3s install

```bash
ssh-copy-id vagrant@172.17.8.101

k3sup install \
  --ip=172.17.8.101 \
  --user=vagrant \
  --sudo \
  --tls-san=172.17.8.100 \
  --cluster \
  --k3s-channel=stable \
  --k3s-extra-args "--no-deploy=traefik --flannel-iface=enp0s8 --node-ip=172.17.8.101" \
  --merge \
  --local-path $HOME/.kube/config \
  --context=k3s-ha
```

### Install vault

```bash
sudo dnf install -y dnf-plugins-core nano jq

cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo dnf install -y vault
```

```bash
nano /etc/vault.d/vault.hcl
# HTTP listener
listener "tcp" {
  address = "0.0.0.0:8200"
  tls_disable = 1
}

# HTTPS listener
#listener "tcp" {
#  address       = "0.0.0.0:8200"
#  tls_cert_file = "/opt/vault/tls/tls.crt"
#  tls_key_file  = "/opt/vault/tls/tls.key"
#}

api_addr         = "http://0.0.0.0:8200"
```

```bash
systemctl start vault
systemctl enable vault

vault -autocomplete-install
complete -C /usr/bin/vault vault

export VAULT_ADDR=http://127.0.0.1:8200
echo "export VAULT_ADDR=http://127.0.0.1:8200" >> ~/.bashrc

vault status

vault operator init | tee /opt/vault/init.txt
Unseal Key 1: t4PsGsw8cj25l9tSpvh2Avr5647HhdaI27aAzSiYJz0=

Initial Root Token: s.sPKauYvv9iFKliclTIaMgbU1
```

```bash
export VAULT_TOKEN="s.sPKauYvv9iFKliclTIaMgbU1"

vault operator unseal t4PsGsw8cj25l9tSpvh2Avr5647HhdaI27aAzSiYJz0=
```

```bash
vault auth enable userpass
vault write auth/userpass/users/devopstales \
    password=Password1 \
    policies=admins
```

### Integrate a Kubernetes Cluster with an External Vault

https://3p8owy1gdkoh452nrc36wbnp-wpengine.netdna-ssl.com/wp-content/uploads/2018/12/nirmata-vault-7-1024x623.png

```bash
vault secrets enable kv

vault kv put kv/secret/devwebapp/config username='giraffe' password='salsa'

vault kv get -format=json kv/secret/devwebapp/config | jq ".data"
```

```bash
# create policy
vault policy write devwebapp-kv-ro - <<EOF
path "kv/secret/devwebapp/*" {
    capabilities = ["read", "list"]
}
EOF
```


```bash
# create Kubernetes ServiceAccount
cat > internal-app.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: devwebapp
---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: role-tokenreview-binding
    namespace: default
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: system:auth-delegator
  subjects:
  - kind: ServiceAccount
    name: devwebapp
    namespace: default
EOF

kubectl apply --filename internal-app.yaml
```


```bash
export EXTERNAL_VAULT_ADDR=172.17.8.101
export K8S_HOST=172.17.8.101

export VAULT_SA_NAME=$(kubectl get sa devwebapp \
    -o jsonpath="{.secrets[*]['name']}")

export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME \
    -o jsonpath="{.data.token}" | base64 --decode; echo)

export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME \
    -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)

vault auth enable kubernetes

vault write auth/kubernetes/config \
  issuer="https://kubernetes.default.svc.cluster.local" \
  token_reviewer_jwt="$SA_JWT_TOKEN" \
  kubernetes_host="https://$K8S_HOST:6443" \
  kubernetes_ca_cert="$SA_CA_CRT"

vault write auth/kubernetes/role/devwebapp \
        bound_service_account_names=devwebapp \
        bound_service_account_namespaces=default \
        policies=devwebapp-kv-ro \
        ttl=24h
```

### INstall Vault Agent Injector 

```bash
dnf copr enable cerenit/helm -y
dnf install helm -y

helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

### változók???
helm install vault hashicorp/vault \
    --set "injector.externalVaultAddr=http://$EXTERNAL_VAULT_ADDR:8200"
```

### Demo

```bash
cat > devwebapp.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orgchart
  labels:
    app: orgchart
spec:
  selector:
    matchLabels:
      app: orgchart
  replicas: 1
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "devwebapp"
        vault.hashicorp.com/agent-inject-secret-config.txt: "kv/secret/devwebapp/config"
      labels:
        app: orgchart
    spec:
      serviceAccountName: devwebapp
      containers:
        - name: orgchart
          image: jweissig/app:0.0.1
EOF

kubectl apply -f devwebapp.yaml

kubectl exec \
    $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") \
    --container orgchart -- cat /vault/secrets/config.txt
```

