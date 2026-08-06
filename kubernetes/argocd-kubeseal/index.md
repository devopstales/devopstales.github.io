---
title: ArgoCD and kubeseal to encrypt secrets
url: https://devopstales.github.io/kubernetes/argocd-kubeseal/
date: 2021-04-10
keywords: kubernetes deployment, gitops, Argo CD, kubeseal
---


In this post I will show you how you can use kubeseal with ArgoCD to protect secrets.

<!--more-->

{{< content "/filedir/k8s-gitops.html" >}}

## Install Argocd

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# or in ha
kubectl apply -n argocd -f \
https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml
```

### Install Argocd cli

```bash
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')

curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64

chmod +x /usr/local/bin/argocd

argocd version
```

### Connect without ingress

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Create ingress for server

```yaml
---
piVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-http-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    cert-manager.io/cluster-issuer: ca-issuer
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: http
    host: argocd.k8s.intra
  tls:
  - hosts:
    - argocd.k8s.intra
    secretName: https-argocd-secret # do not change, this is provided by Argo CD
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-grpc-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: https
    host: grpc-argocd.k8s.intra
  tls:
  - hosts:
    - grpc-argocd.k8s.intra
    secretName: argocd-secret # do not change, this is provided by Argo CD
```

### Get init password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
jMnyrjcdocMoqPfC

argocd login argocd.k8s.intra
WARNING: server certificate had error: x509: certificate signed by unknown authority. Proceed insecurely (y/n)? y
WARN[0002] Failed to invoke grpc call. Use flag --grpc-web in grpc calls. To avoid this warning message, use flag --grpc-web. 
Username: admin
Password: 
'admin:login' logged in successfully
Context 'argocd.k8s.intra' updated

argocd account update-password
```

### Register new cluster
By default Argocd register the cluster where running with a cluster admin service account. You can register different clusters with its kubectl configs. So you can create multiple service accounts with multiple privileges  and register them with its kubectl configs.

```bash
$ kubectx
default

$ argocd cluster add default
INFO[0000] ServiceAccount "argocd-manager" already exists in namespace "kube-system" 
INFO[0000] ClusterRole "argocd-manager-role" updated    
INFO[0000] ClusterRoleBinding "argocd-manager-role-binding" updated 
WARN[0000] Failed to invoke grpc call. Use flag --grpc-web in grpc calls. To avoid this warning message, use flag --grpc-web. 
Cluster 'https://172.17.9.10:6443' added
```

### Deploy app with Argocd

```bash
$ kubectl create ns guestbook-demo

$ argocd app create 00-tools --repo https://github.com/devopstales/gitops-repo.git \
--path 00_argocd/00_tools --dest-server https://kubernetes.default.svc --dest-namespace default

$ argocd app create 01-guestbook --repo https://github.com/devopstales/gitops-repo.git \
--path 00_argocd/01_guestbook --dest-server https://kubernetes.default.svc --dest-namespace guestbook-demo

$ argocd app get 01-guestbook
Name:               01-guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          guestbook-demo
URL:                https://argocd.k8s.intra/applications/01-guestbook
Repo:               https://github.com/devopstales/gitops-repo.git
Target:             
Path:               00_argocd/01_guestbook
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        OutOfSync from  (e8df0a5)
Health Status:      Missing

GROUP                      KIND         NAMESPACE       NAME                            STATUS     HEALTH   HOOK  MESSAGE
                           Service      guestbook-demo  guestbook-ui                    OutOfSync  Missing        
apps                       Deployment   guestbook-demo  guestbook-ui                    OutOfSync  Missing        
rbac.authorization.k8s.io  RoleBinding  guestbook-demo  psp-rolebinding-guestbook-demo  OutOfSync  Missing   

$ argocd app sync 01-guestbook

```

## Install kubeseal

```bash
VERSION=$(curl --silent "https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')

wget https://github.com/bitnami-labs/sealed-secrets/releases/download/$VERSION/kubeseal-linux-amd64 -O /usr/local/bin/kubeseal
chmod 755 /usr/local/bin/kubeseal
kubeseal --version
```

### Now install the cluster-side controller

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/$VERSION/controller.yaml
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

Since the secrets aren't encrypted, it is unsecure to commit them to your Git repository.

### Use kubeseal to Encrypt the secret

```yaml
kubeseal --format yaml <secret.yaml >sealedsecret.yaml

cat sealedsecret.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: mysecret
  namespace: demo-app
spec:
  encryptedData:
    secret: AgCZl5b75Lmr8z7Ppa5tNBPX4zWH2vset0GVKhNfTBRnAANDs9Gycrq5EfueE00PGX++v4VFKEwi9rNeAAvFkETges31Uhi4+Oym9CkV9rU2pHAvD4iZapt+fHSndMUY8vWT8GCYzrzSOFSRPKB4cdAy3JJ4f48SwxCFYXdJgl/6KiHkrk2AzxHKip3ryVjKY01E8cSpxw1Exv8RnEDD8D9hfb57fEIRRwMrIRUkg/jPOvf4YCHcjHiVLLP+MwutT1Jd65hjAx1WZFSjDRUj3rFfzsO6zAVxgx20WXtc3qMK9jMeeQaNbbAvdv3YuNsuxJIE8SFQFPfGop+QFefiyDGWTjzwHkeU65Ci1Nuj8pSS600ITyGdyNY4F3qjen1eBnMOaub5ZJqEmXyTQwSL/9R7UfoFqJCo4b36g2axacegqHtLL+U4wrHsDB9iQ/JrEAWj4l7s5bhOJbq0N8zLwZvEGXSoPs/4eBUxCuHayOCz6o8BY8Zsv1tDgQ+AXpvudXfzw02zH/DCr7Jg2CVXB8Qk2SUnC5rMzsvqcsYnHP25pxGh9qd3p8QXIjb+AttJUFkPGHlc/rY6sY4QJ6Qjlfv8VXArwrmnfkcZSfLDwyUOGcqZiho3+vGC4mjDcFgbEDbD3Emv/2jHimFBOv2eq9dMqvmZuzk4M4KCLYHqFuX+L/XM+mAnAxlCRrv6q6Hup26HuI84Hn2N
  template:
    metadata:
      creationTimestamp: null
      name: mysecret
      namespace: demo-app
```

`sealedsecret.yaml` is the file you need to store in git.

```bash
argocd app create 02-secret --repo https://github.com/devopstales/gitops-repo.git \
--path 00_argocd/02_secret --dest-server https://kubernetes.default.svc --dest-namespace demo-app

kubectl logs -n demo-app demo-app
S3CR3T
```

