---
title: Kubernetes project longhorn
url: https://devopstales.github.io/kubernetes/k8s-longhorn/
date: 2020-01-18
keywords: Kubernetes, k8s, project longhorn, Rancher
---


Longhorn is lightweight, reliable, and powerful distributed block storage system for Kubernetes..
<!--more-->

{{< content "/filedir/kubernetes.html" >}}

You can install Longhorn on an existing Kubernetes cluster with one `kubectl apply` command or using Helm charts. Once Longhorn is installed, it adds persistent volume support to the Kubernetes cluster.


### Install dependency
```
yum install iscsi-initiator-utils

modprobe iscsi_tcp
echo "iscsi_tcp" >/etc/modules-load.d/iscsi-tcp.conf
```

### Deploy Longhorn and storageclass
```
kubectl apply -f https://raw.githubusercontent.com/rancher/longhorn/master/deploy/longhorn.yaml
kubectl apply -f https://raw.githubusercontent.com/rancher/longhorn/master/examples/storageclass.yaml

kubectl get storageclass
NAME       PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn   driver.longhorn.io   Delete          Immediate           false                  11m
```

Patch longhorn storageclass to be the default storageclass.

```
kubectl patch storageclass longhorn -p \
  '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl get storageclass
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn (default)   driver.longhorn.io   Delete          Immediate           false                  12m
```

### Deploy admin gui
```
kubectl -n longhorn-system get svc

cat minio-sec.yaml
---
apiVersion: v1
kind: Secret
metadata:
  namespace: longhorn-system
  name: longhorn-minio
type: Opaque
data:
  AWS_ACCESS_KEY_ID: bWluaW8=
  AWS_SECRET_ACCESS_KEY: bWluaW8xMjM=
  AWS_ENDPOINTS: aHR0cDovL21pbmlvLmxvbmdob3JuLXN5c3RlbTo5MDAw
```

![Example image](/img/include/longhorn0.webp)</br>
![Example image](/img/include/longhorn2.webp)</br>
![Example image](/img/include/longhorn1.webp)</br>

