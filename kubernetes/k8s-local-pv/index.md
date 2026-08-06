---
title: Kubernetes nginx ingress with helm
url: https://devopstales.github.io/kubernetes/k8s-local-pv/
date: 2020-01-08
keywords: kubernetes deployment
---


In this post I will show you how to use a local folder as a persistent volume in Kubernetes.
<!--more-->

{{< content "/filedir/kubernetes.html" >}}

For a production environment this is not an ideal structure because if you store the data on a single host if the host dies your data will be lost. For this Demo I will use a separate disk for storing the PV's folders. So you can backup or replicate this disk separately.

### Configure the disk
```
vgcreate local-vg /dev/sdd
lvcreate -l 100%FREE -n local-lv local-vg /dev/sdd
mkfs.xfs -f /dev/local-vg/local-lv
mkdir -p /mnt/local-storage/
mount /dev/local-vg/local-lv /mnt/local-storage
echo "/dev/local-vg/local-lv        /mnt/local-storage              xfs defaults 0 0" >> /etc/fstab
rm -rf /mnt/local-storage/lost+found
```

Now you can create every PV and PVC manually.
```
mkdir /mnt/local-storage/pv-tst

cat pv-tst.yaml
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-tst
spec:
  capacity:
    storage: 1Gi
  local:
    path: /mnt/local-storage/pv-tst
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - kubernetes03.devopstales.intra
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-tst
  namespace: tst
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeName: pv-tst
  storageClassName: local
```

### Add automated hostpath-provisioner
This is a Persistent Volume Claim (PVC) provisioner for Kubernetes. It dynamically provisions hostPath volumes to provide storage for PVCs.
```
git clone https://github.com/torchbox/k8s-hostpath-provisioner
cd k8s-hostpath-provisioner
kubectl apply -f deployment.yaml

nano local-sc.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: auto-local
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: torchbox.com/hostpath
parameters:
  pvDir: /mnt/local-storage
```

Test the provisioner by creating a new PVC:
```
cat testpvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: testpvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 50Gi

kubectl create -f testpvc.yaml
kubectl get pvc
NAME      STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
testpvc   Bound     pvc-145c785e-ab83-11e7-9432-4201ac1fd019   50Gi       RWX            auto-local     10s
```

