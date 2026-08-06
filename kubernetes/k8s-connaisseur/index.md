---
title: Image Signature Verification Admission Controller
url: https://devopstales.github.io/kubernetes/k8s-connaisseur/
date: 2021-02-22
keywords: kubernetes deployment, kubernetes vs docker, Connaisseur, Notary
---


In this post I will show you how you can deploy Connaisseur to Image Signature Verification into a Kubernetes cluster.

<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

### What is Connaisseur?
Connaisseur is an admission controller for Kubernetes that integrates Image Signature Verification into a cluster, as a means to ensure that only valid images are being deployed.


### Notary
Notary is an open source signing solution for containers based on The Update Framework Notary uses TUFs’ roles and key hierarchy for signing of the images. There are five keys to sign the metadata files which lists all filenames in the collection, their sizes and respective hashes.

```bash
apt install notary
```

```bash
docker pull alpine
docker tag alpine:latest devopstales/testimage:unsigned
docker push devopstales/testimage:unsigned
```


```bash
notary -s https://notary.docker.io -d ~/.docker/trust init -p docker.io/devopstales/testimage     
Root key found, using: 31579f2a034add499da6e799bc9260d08a15ab1804298218f05f78d97a669f77
Enter passphrase for root key with ID 31579f2: 
Enter passphrase for new targets key with ID 42e49c6: 
Repeat passphrase for new targets key with ID 42e49c6: 
Enter passphrase for new snapshot key with ID 399243c: 
Repeat passphrase for new snapshot key with ID 399243c: 
Enter username: devopstales
Enter password: 
Auto-publishing changes to docker.io/devopstales/testimage
Enter username: devopstales
Enter password: 
Successfully published changes for repository docker.io/devopstales/testimage


export DOCKER_CONTENT_TRUST=1
export DOCKER_CONTENT_TRUST_SERVER=https://notary.docker.io
docker tag alpine:latest devopstales/testimage:signed
docker push devopstales/testimage:signed
```

```bash
$ find ~/.docker/trust/ | head
/home/devopstales/.docker/trust/
/home/devopstales/.docker/trust/private
/home/devopstales/.docker/trust/private/1f4a9a0922605b3bc19c97e180d962d530721288f4fd0845ad0aa37ba4a6f95d.key
/home/devopstales/.docker/trust/private/fe30e72f5976b2ae7d0d365f28dacfae9c71f11ad854065603ccc806900e84fa.key
/home/devopstales/.docker/trust/private/3da0d27e2d3b964d238d1d184c7578b5f2737b918ec5b8265474e22b07b2ea22.key
/home/devopstales/.docker/trust/private/root-priv.key
/home/devopstales/.docker/trust/private/root-pub.pem
/home/devopstales/.docker/trust/tuf
/home/devopstales/.docker/trust/tuf/docker.io
/home/devopstales/.docker/trust/tuf/docker.io/devopstales
```

```bash
notary -s https://notary.docker.io -d ~/.docker/trust list docker.io/devopstales/testimage
NAME     DIGEST                                                              SIZE (BYTES)    ROLE
----     ------                                                              ------------    ----
signed    4661fb57f7890b9145907a1fe2555091d333ff3d28db86c3bb906f6a2be93c87    528             targets/devopstales
```

### Install Connaisseur

```bash
# The installer use yq so we need to install it

wget https://github.com/mikefarah/yq/releases/download/v4.2.0/yq_linux_amd64 -O /usr/bin/yq &&\
    chmod +x /usr/bin/yq
```

```bash
# generate the public root cert

cd ~/.docker/trust/private
sed '/^role:\sroot$/d' $(grep -iRl "role: root" .) > root-priv.key
openssl ec -in root-priv.key -pubout -out root-pub.pem
```

```yaml
git clone https://github.com/sse-secure-systems/connaisseur.git
cd connaisseur
nano helm/values.yaml
...
# the public part of the root key, for verifying notary's signatures
  rootPubKey: |
    -----BEGIN PUBLIC KEY-----
    MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE9m6WfwViwT8lYjLF6jAs1bvd1hPp
    cRUmONP49JszW1X/6Q22DygylIJGyC8IXeb3zBWVMoYDxauiqrFomHUOEA==
    -----END PUBLIC KEY-----

make install
```

### Test the Image Signature Verification

```bash
kubens default

kubectl run unsigned --image=docker.io/devopstales/testimage:unsigned
Error from server: admission webhook "connaisseur-svc.connaisseur.svc" denied the request: failed to verify signature of trust data.

kubectl run signed --image=docker.io/devopstales/testimage:signed
pod/signed created

kubectl get po
```

### Final words

Connaisseur is a grate tool but has a few shortcomings:

* There is no option to whitelist images in a specific namespace.
* Connaisseur supports only one Notary server
* Connaisseur supports only one public key

#### Update:
[Connaisseur 2.0](/kubernetes/k8s-connaisseur-v2) is released and solve all of this problems.

