---
title: Kubernetes CephFS volume with CSI driver
url: https://devopstales.github.io/kubernetes/k8s-cephfs-storage-with-csi-driver/
date: 2023-05-30
keywords: rook, ceph, cephfs, kubernetes, deployment
---


In this post I will show you how can you use CephFS with CSI driver for persistent storage on Kubernetes.

<!--more-->

{{< content "/filedir/kubernetes.html" >}}

The Container Storage Interface (CSI) is a standard for exposing arbitrary block and file storage storage systems to Kubernetes. Using CSI third-party storage providers can write and deploy plugins exposing storage systems in Kubernetes. Before we begin lets ensure that we have the following requirements:

* Kubernetes cluster v1.14+
* allow-privileged flag enabled for both kubelet and API server
* Running Ceph cluster
* Created CephFS

First we need to create a namespace for the storage provider:

```bash
kubectl create ns ceph-csi-cephfs
kubens ceph-csi-cephfs
```

Login to the CEPH cluster and get the configurations:

```bash
ceph config generate-minimal-conf > ceph-minimal.conf

cat ceph-minimal.conf
# minimal ceph.conf for e285a458-7c95-4187-8129-fbd6c370c537
[global]
    fsid = e285a458-7c95-4187-8129-fbd6c370c537
    mon_host = [v2:192.168.10.11:3300/0,v1:192.168.10.11:6789/0] [v2:192.168.10.12:3300/0,v1:192.168.10.12:6789/0] [v2:192.168.10.13:3300/0,v1:192.168.10.13:6789/0]

ceph auth get-key client.admin
QVFDWDNuVmtNV3NvSlJBQUFvazIxMCszZXFxNmF6SmpT5WJjaUE9PQ==
```

```bash
helm repo add ceph-csi https://ceph.github.io/csi-charts
helm show values ceph-csi/ceph-csi-cephfs > defaultValues.yaml
```

Crate helm values file:

```yaml
nano values.yaml
---
csiConfig:
  - clusterID: e285a458-7c95-4187-8129-fbd6c370c537
    monitors:
      - 192.168.10.11:6789
      - 192.168.10.12:6789
      - 192.168.10.13:6789
    cephFS:
      subvolumeGroup: "csi"
secret:
  name: csi-cephfs-secret
  adminID: admin
  adminKey: QVFDWDNuVmtNV3NvSlJBQUFvazIxMCszZXFxNmF6SmpT5WJjaUE9PQ==
  create: true
storageClass:
  create: true
  name: k8s-cephfs
  clusterID: e285a458-7c95-4187-8129-fbd6c370c537
  # (required) CephFS filesystem name into which the volume shall be created
  fsName: k8s-etc-nvme
  reclaimPolicy: Delete
  allowVolumeExpansion: true
  volumeNamePrefix: "poc-k8s-"
  provisionerSecret: csi-cephfs-secret
  controllerExpandSecret: csi-cephfs-secret
  nodeStageSecret: csi-cephfs-secret
```

Deploy helm chart:

```bash
helm upgrade --install ceph-csi-cephfs ceph-csi/ceph-csi-cephfs --values ./values.yaml
```

### Demo time

```yaml
nano pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-cephfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-cephfs-sc
```

```bash
kubectl apply -f pvc.yaml


kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
csi-cephfs-pvc   Bound    pvc-51526639-6fef-4abd-b453-c2b03c08781f   1Gi        RWX            csi-cephfs-sc   31m
```

