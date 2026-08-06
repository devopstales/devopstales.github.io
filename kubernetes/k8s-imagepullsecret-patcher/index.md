---
title: How to use imagePullSecrets cluster-wide??
url: https://devopstales.github.io/kubernetes/k8s-imagepullsecret-patcher/
date: 2021-02-17
keywords: kubernetes deployment, imagePullSecrets, Dockerhub limit, Kyverno
---


In this post I will show you how you can use imagePullSecrets cluster-wide in Kubernetes.
<!--more-->

Kubernetes uses imagePullSecrets to authenticate to private container registris on a per Pod or per Namespace basis. To do that yo need to create a secret with the credentials:

```bash
kubectl create secret docker-registry image-pull-secret \
  -n <your-namespace> \
  --docker-server=<your-registry-server> \
  --docker-username=<your-name> \
  --docker-password=<your-password> \
  --docker-email=<your-email>
```

Now we can use this secret in a pod for download the docker image:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: private-registry-test
spec:
  containers:
    - name: my-app
      image: my-private-registry.intra/busybox:v1
  imagePullSecrets:
    - name: image-pull-secret
```

The other way is to add it to the default ServiceAccount in the namespace:

```bash
kubectl patch serviceaccount default \
  -p "{\"imagePullSecrets\": [{\"name\": \"image-pull-secret\"}]}" \
  -n <your-namespace>
```

I found a tool called imagepullsecret-patcher that do this on all of your namespace:

```bash
wget https://raw.githubusercontent.com/titansoft-pte-ltd/imagepullsecret-patcher/185aec934bd01fa9b6ade2c44624e5f2023e2784/deploy-example/kubernetes-manifest/1_rbac.yaml
wget https://raw.githubusercontent.com/titansoft-pte-ltd/imagepullsecret-patcher/master/deploy-example/kubernetes-manifest/2_deployment.yaml

kubectl create ns imagepullsecret-patcher
```

Edit the downloaded file and chaneg the contant of the image-pull-secret-src and the namespace if nececary

```bash
nano 1_rbac.yaml
nano 2_deployment.yaml
kubectl apply -f 1_rbac.yaml
kubectl apply -f 2_deployment.yaml
```

### test

```bash
kubectl create ns imagepullsecret-test
kubectl get secret image-pull-secret -n imagepullsecret-test
image-pull-secret   kubernetes.io/dockerconfigjson   1      9m35s
```

The secret is automaticle created.

### Kyverno policy
You can do the same thing with kyverno policy:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: sync-secret
spec:
  background: false
  rules:
  - name: sync-image-pull-secret
    match:
      resources:
        kinds:
        - Namespace
    generate:
      kind: Secret
      name: image-pull-secret
      namespace: "{{request.object.metadata.name}}"
      synchronize: true
      clone:
        namespace: default
        name: image-pull-secret
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: mutate-imagepullsecret
spec:
  rules:
    - name: mutate-imagepullsecret
      match:
        resources:
          kinds:
          - Pod
      mutate:
        patchStrategicMerge:
          spec:
            imagePullSecrets:
            - name: image-pull-secret  ## imagePullSecret that you created with docker hub pro account
            (containers):
            - (image): "*" ## match all container images
```

