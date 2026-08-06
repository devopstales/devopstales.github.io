---
title: How to create kubeconfig?
url: https://devopstales.github.io/kubernetes/k8s-rbac-gen/
date: 2021-11-03
keywords: Keycloak, Kubernetes, service account, kubeconfig, RBAC
---


In this blog, I will show you how to create a kubeconfig file with limited access to kubernetes cluster using service account, secret token and RBAC

<!--more-->

Create namespace:

```bash
export NAMESPACE=test-ns
export SERVICEACCOUNT=devopstales

kubectl create namespace $NAMESPACE
kubens $NAMESPACE
```

Create serviceaccount with RBAC:

```yaml
cat <<EOF | envsubst | kubectl create -f -
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: $SERVICEACCOUNT
  namespace: $NAMESPACE
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: developer-access
  namespace: $NAMESPACE
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["*"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: $SERVICEACCOUNT
  namespace: $NAMESPACE
subjects:
- kind: ServiceAccount
  name: $SERVICEACCOUNT
  namespace: $NAMESPACE
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer-access
EOF
```

Create kubeconfig for serviceaccount:

```bash
git clone https://github.com/devopstales/k8s_sec_lab.git
cd k8s_sec_lab/kubernetes-scripts
chmod +x create-kubeconfig.sh
./create-kubeconfig.sh $SERVICEACCOUNT > kubeconfig-$NAMESPACE
```

Use kubeconfig:

```bash
kubectl --kubeconfig=kubeconfig-$NAMESPACE get po
```

### Permission Managger
Permission Manager is an application developed by SIGHUP that enables a super-easy and user-friendly RBAC management for Kubernetes.


```bash
kubectl create namespace permission-manager
```

```yaml
nano pm-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: permission-manager
  namespace: permission-manager
type: Opaque
stringData:
  PORT: "4000" # port where server is exposed
  CLUSTER_NAME: "my-cluster" # name of the cluster to use in the generated kubeconfig file
  CONTROL_PLANE_ADDRESS: "https://172.17.0.3:6443" # full address of the control plane to use in the generated kubeconfig file
  BASIC_AUTH_PASSWORD: "changeMe" # password used by basic auth (username is `admin`)
```

Deploy permission-manager:

```bash
kubectl apply -f pm-secret.yaml
kubectl apply -f https://github.com/sighupio/permission-manager/releases/download/v1.7.1-rc1/crd.yml
kubectl apply -f https://github.com/sighupio/permission-manager/releases/download/v1.7.1-rc1/seed.yml
kubectl apply -f https://github.com/sighupio/permission-manager/releases/download/v1.7.1-rc1/deploy.yml

kubectl port-forward svc/permission-manager 4000 --namespace permission-manager
```

Connect on localhost:

![permission-manager](/img/include/permission-manager.webp)
