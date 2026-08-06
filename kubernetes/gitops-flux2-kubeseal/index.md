---
title: Flux2 and kubeseal to encrypt secrets
url: https://devopstales.github.io/kubernetes/gitops-flux2-kubeseal/
date: 2021-05-07
keywords: kubernetes deployment, gitops, Flux2, kubeseal, Sealed Secrets, Mozilla SOPS
---


In this post I will show you how you can use kubeseal and Mozilla SOPS with Flux2 to protect secrets.

<!--more-->

{{< content "/filedir/k8s-gitops.html" >}}

## Install kubeseal cli

```bash
VERSION=$(curl --silent "https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')

wget https://github.com/bitnami-labs/sealed-secrets/releases/download/$VERSION/kubeseal-linux-amd64 -O /usr/local/bin/kubeseal
chmod 755 /usr/local/bin/kubeseal
kubeseal --version
```

### Deploy sealed-secrets with a HelmRelease

```bash
flux create source helm sealed-secrets \
--interval=1h \
--url=https://bitnami-labs.github.io/sealed-secrets

flux create helmrelease sealed-secrets \
--interval=1h \
--release-name=sealed-secrets \
--target-namespace=flux-system \
--source=HelmRepository/sealed-secrets \
--chart=sealed-secrets \
--chart-version=">=1.15.0-0" \
--crds=CreateReplace
```

At startup, the sealed-secrets controller generates a 4096-bit RSA key pair and persists the private and public keys as Kubernetes secrets in the flux-system namespace.

```bash
kubeseal --fetch-cert \
--controller-name=sealed-secrets \
--controller-namespace=flux-system \
> ../pub-sealed-secrets.pem
```

Generate a Kubernetes secret manifest with kubectl:

```bash
kubectl -n default create secret generic basic-auth \
--from-literal=user=admin \
--from-literal=password=change-me \
--dry-run=client \
-o yaml > ../basic-auth.yaml
```

### Create a sealed secret

Create a secret you want to encrypt:

```yaml
apiVersion: v1
data:
  secret: UzNDUjNUCg==
kind: Secret
metadata:
  creationTimestamp: null
  name: mysecret
  namespace: demo-app
```

A secret in Kubernetes cluster is encoded in base64 but not encrypted! Theses data are "only" encoded so if a user have access to your secrets, he can simply base64 decode to see your sensitive data:

```base
echo "UzNDUjNUCg==" | base64 -d
S3CR3T
```

Encrypt the secret with kubeseal:

```bash
mkdir ./01_flux2/02_secret/

kubeseal --format yaml --cert=../../../pub-sealed-secrets.pem \
< secret.yaml >sealedsecret.yaml
rm -f secret.yaml

git add -A
git commit -m "kubeseal"
git push

kubectl logs -n demo-app demo-app
S3CR3T
```

