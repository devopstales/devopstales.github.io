---
title: Flux2 and Mozilla SOPS to encrypt secrets
url: https://devopstales.github.io/kubernetes/gitops-flux2-sops/
date: 2021-05-08
keywords: kubernetes deployment, gitops, Flux2, Sealed Secrets, Mozilla SOPS
---


In this post I will show you how you can use Mozilla SOPS with Flux2 to protect secrets.

<!--more-->

{{< content "/filedir/k8s-gitops.html" >}}

First you need to bootstrap the fluxcomponent as I showd in the [previous post](../gitops-flux2/).

## Install OPS CLI

```bash
VERSION=$(curl --silent "https://api.github.com/repos/mozilla/sops/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
VERSION=${VERSION:1}

wget https://github.com/mozilla/sops/releases/download/v$VERSION/sops-"$VERSION"-1.x86_64.rpm
yum install -y sops-"$VERSION"-1.x86_64.rpm
rm -f sops-"$VERSION"-1.x86_64.rpm
```

### Generate a GPG key

Generate a GPG/OpenPGP key:
```bash
export KEY_NAME="cl1.mydomain.intra"
export KEY_COMMENT="flux secrets"

gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Comment: ${KEY_COMMENT}
Name-Real: ${KEY_NAME}
EOF
```

The above configuration creates an rsa4096 key that does not expire. Retrieve the GPG key fingerprint:
```bash
gpg --list-secret-keys "${KEY_NAME}"

sec   rsa4096 2020-09-06 [SC]
      1F3D1CED2F865F5E59CA564553241F147E7C5FA4

# Store the key fingerprint as an environment variable:
export KEY_FP=1F3D1CED2F865F5E59CA564553241F147E7C5FA4
```

Export the public and private key from your local GPG keyring and create a Kubernetes secret named sops-gpg in the flux-system namespace:

```bash
gpg --export-secret-keys --armor "${KEY_FP}" |
kubectl create secret generic sops-gpg \
--namespace=flux-system \
--from-file=sops.asc=/dev/stdin

mkdir ./02_flux2/03_SOPS_demo/
gpg --export-secret-keys --armor "${KEY_FP}" > ./02_flux2/03_SOPS_demo/sops.pub.asc
```

It’s a good idea to back up this secret-key/K8s-Secret with a password manager or offline storage. Also consider deleting the secret decryption key from you machine:

```bash
gpg --delete-secret-keys "${KEY_FP}"
```

```bash
git add -A
git commit -m "add sops config"
git push
```

### Configure in-cluster secrets decryption

```bash
nano 02_flux2/flux-system/gotk-sync.yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: flux-system
  url: ssh://git@github.com/devopstales/gitops-repo
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-gpg
  interval: 10m0s
  path: ./01_flux2
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  validation: client
```

Create a secret you want to encrypt:

```yaml
cp 01_flux2/02_secret/*.yaml 02_flux2/03_SOPS_demo/
rm -f 02_flux2/03_SOPS_demo/sealedsecret.yaml

nano 02_flux2/03_SOPS_demo/secret.yaml
---
apiVersion: v1
data:
  secret: UzNDUjNUCg==
kind: Secret
metadata:
  creationTimestamp: null
  name: mysecret
  namespace: demo-app
```

ncrypt the secret with SOPS using your GPG key:

```bash
sops --encrypt --encrypted-regex '^(data|stringData)$' --pgp ${KEY_FP} \
--in-place 02_flux2/03_SOPS_demo/secret.yaml
```

Check the result:

```yaml
cat 02_flux2/03_SOPS_demo/secret.yaml 
apiVersion: v1
data:
    secret: ENC[AES256_GCM,data:RxdPIf8i5G+yjiT0,iv:A8iYsJ4hdd1MNZDAKQyD4L/b6Caa1TDn+MNKsFku3nc=,tag:QLN3Y6B/S87qzPYh2PATaQ==,type:str]
kind: Secret
metadata:
    creationTimestamp: null
    name: mysecret
    namespace: demo-app
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age: []
    lastmodified: "2021-05-09T08:59:30Z"
    mac: ENC[AES256_GCM,data:2SwDXqI5R880Y/uf4yW6o3rraJ7WYQ5aIfKwIPEpss/evD15dfLulUUahN0bNmrwtSYuad0aXHopGfFsKEvxSVMx9UkPaAd0xVZHSj7aJ5Fi5D2De3Tw1yi8tuWjU8OMG81nIZkx6GdIa4yZlm+LEangYVIpzluWk6C7In8GNc8=,iv:9E0R+R6QjqCXBnzPlAzri/fYxEr0HPZ0FzQX5oddMZk=,tag:FEJknbbaqp402biU/hxUaA==,type:str]
    pgp:
        - created_at: "2021-05-09T08:59:30Z"
          enc: |
            -----BEGIN PGP MESSAGE-----

            hQIMA1gFjmLlSkpZAQ//ZwU9ZEL2jDtmg8ZvJ4Wrnaa66Rjvnr/Uz9VOUxKZk1WJ
            V6+Wa1Od80tODzr9gfmjHor0ZCbdJmPxf96z4MhcBNbo9oBr43GX2Wm67ijwIEdo
            Wup2252ANsn1stZIk5krdlZVkRTW+GeAwEDHnW4zSOSVfc9Ad1SHGy1vgXM/i7Je
            ttx3s/PZhPPZLUQ6SKQjEaf6Xod9nnLyqSb1SdB5idhBy/wV3nt08p/LJ8QVItzh
            iFclOBRXeEQuYEt2O57Bo/kRGTaBjq6YE0KCbu7PMkm8gerZOyW2Od1maF4Bsjlz
            c+qOaLjFOP4K/8OIDtzTOw9tXbQREWC9tl1ReadGHReTbdCs55msmMWscPJtq8wi
            a3eMdLDwvJHhAERaJwvAa5Le6uIwr+lOEVzetj2ucK6LgVlTjgs9IPTMul4ASyji
            tOtTUzXoEHu/wfGKP7QFDbROFBWalNBkSegdOQx+/GSLebfWY/HmTy+V7isRhoa3
            iH4/hapCDUQVqwQSyvjpVUzoAsp9g7XaYITKGbSkIuUA3TI5aTp3SSF7sbNHdG8g
            6i3jh4FxS9yzFgM7fGnlbHDta/DzyBQB5Z77cI8pJW9mri1/U6R63zJuDqSUvFlw
            zf1zJwEV3xgwMog/7nx4aAItPBeqsT0pSYs5pQpciKgTJcCOTG8r6+3+cjckS/vS
            XgH6CQtPyzD1JAVDz94n0RkZKC+TXUfLnRF0yLwaAeOJ9h6vFVnEggOIXiWF4Iy7
            Ds1zIaq79+H/gWo0fGk2srKIHdcPZkQc7zAOqbO5X94fo0TtNx5y/QT6FUwUH4k=
            =GMAU
            -----END PGP MESSAGE-----
          fp: FCF6E84DC263BDDB9A35D3C6DF8C27FF0C09F771
    encrypted_regex: ^(data|stringData)$
    version: 3.7.1
```

Upload files and test:

```bash
git add -A
git commit -m "sops demo"
git push

kubectl logs -n demo-app2 demo-app
S3CR3T
```

