---
title: Openshift: External registry
url: https://devopstales.github.io/kubernetes/openshift-extregistry/
date: 2019-04-19
keywords: ansible, openshift 3.11, openshift okd, openshift tutorial, openshift container platform, rad hat openshift
---


I this post I will demonstrate howyou can use an external registry in Openshift.

<!--more-->

{{< content "/filedir/openshift.html" >}}

### Configuring Openshift
From your client machine, create a Kubernetes secret object for Harbor.:

```
oc new-proyect registrytest

oc create secret docker-registry harbor \
--docker-server=https://harbor.devopstales.intra \
--docker-username=admin \
--docker-email=admin@devopstales.intra \
--docker-password='[your_admin_harbor_password]'
```

If you want you can add this secret to the deafult template of the project creation.

### Deploy the private image on the Openshift cluster
```
nano registrytest-deployment.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: harbor.devopstales.intra/test/nginx:V2
        name: nginx
      imagePullSecrets:
      - name: harbor

oc apply -f kuard-deployment.yaml
oc get pods
```

