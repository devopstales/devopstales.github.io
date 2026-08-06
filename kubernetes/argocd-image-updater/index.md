---
title: Argo CD Image Updater for automate image update
url: https://devopstales.github.io/kubernetes/argocd-image-updater/
date: 2021-04-11
keywords: kubernetes deployment, gitops, Argo CD
---


In this post I will show you how you can use Argo CD Image Updater to automate image update in Kubernetes.

<!--more-->

{{< content "/filedir/k8s-gitops.html" >}}


### What is Argo CD Image Updater
A tool to automatically update the container images of Kubernetes workloads that are managed by Argo CD.

## Inatall Argo CD Image Updater

```bash
VERSION=$(curl --silent "https://api.github.com/repos/argoproj-labs/argocd-image-updater/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')

wget https://github.com/argoproj-labs/argocd-image-updater/releases/download/$VERSION/argocd-image-updater_"$VERSION"_linux-amd64 -O /usr/local/bin/argocd-image-updater
chmod 755 /usr/local/bin/argocd-image-updater

argocd-image-updater version
```

### Now install the cluster-side controller
```bash
cd /opt
git clone https://github.com/argoproj-labs/argocd-image-updater.git
cd argocd-image-updater/manifests/
kubectl apply -f install.yaml
```

### Deploy an updatable app

In order for Argo CD Image Updater to know which applications it should inspect for updating the workloads' container images, the corresponding Kubernetes resource needs to be annotated. or its annotations, Argo CD Image Updater uses the following prefix: `argocd-image-updater.argoproj.io`


```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: gcr.io/heptio-images/ks-guestbook-demo:^0.1
  name: guestbook
  namespace: argocd
spec:
  destination:
    namespace: guestbook-demo
    server: https://kubernetes.default.svc
  project: default
  source:
    path: helm-guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps
    targetRevision: HEAD
```

Test the image for update:

```bash
argocd-image-updater test gcr.io/heptio-images/ks-guestbook-demo:0.1
INFO[0000] getting image                                 image_name=heptio-images/ks-guestbook-demo registry=gcr.io
INFO[0000] Fetching available tags and metadata from registry  image_name=heptio-images/ks-guestbook-demo
INFO[0000] Found 2 tags in registry                      image_name=heptio-images/ks-guestbook-demo
DEBU[0000] found 2 from 2 tags eligible for consideration  image="gcr.io/heptio-images/ks-guestbook-demo:0.1"
INFO[0000] latest image according to constraint is gcr.io/heptio-images/ks-guestbook-demo:0.2
```

Allow update of the image:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: gcr.io/heptio-images/ks-guestbook-demo
    argocd-image-updater.argoproj.io/write-back-method: argocd
  name: guestbook
  namespace: argocd
spec:
  destination:
    namespace: guestbook-demo
    server: https://kubernetes.default.svc
  project: default
  source:
    path: helm-guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps
    targetRevision: HEAD
```

The Argo CD Image Updater supports two distinct methods on how to update images of an application:

* imperative, via Argo CD API
* declarative, by pushing changes to a Git repository

The write-back method is configured via an annotation on the Application resource:

```bash
argocd-image-updater.argoproj.io/write-back-method: <argocd>
# argocd or git

argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd-image-updater/git-creds
# add git credentials secret named git-creds

argocd-image-updater.argoproj.io/git-branch: HEAD
# Specifying a branch to commit to
```

At the gui you can see that the guestbook app is out of sync and can be updated.

