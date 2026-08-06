---
title: Kubernetes volume expansion with Ceph RBD CSI driver
url: https://devopstales.github.io/kubernetes/k8s-ceph-csi-extand/
date: 2022-10-20
keywords: rook, ceph, kubernetes deployment, volume expansion
---


In this post I will show you how can you use CEPH RBD CSI driver as persistent storage end enable volume expansion on Kubernetes.

<!--more-->

{{< content "/filedir/kubernetes.html" >}}


Firt we need to install the [csi-resizer](https://github.com/kubernetes-csi/external-resizer). This is a sidecar container that watches Kubernetes PersistentVolumeClaims objects and triggers controller side expansion operation against a CSI endpoint 

```bash
kubens ceph-csi-rbd
wget https://raw.githubusercontent.com/kubernetes-csi/external-resizer/master/deploy/kubernetes/rbac.yaml

# chaneg namespace to your namespace in my case ceph-csi-rbd
nano rbac.yaml

kubectl apply -f rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-resizer/master/deploy/kubernetes/deployment.yaml
```

Create or edit a StorageClass with the peccary parameters like (`csi.storage.k8s.io/controller-expand-secret-namespace: default`,`csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret`,`allowVolumeExpansion: true`):

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: k8s-etc
  annotations:
    storageclass.kubernetes.io/is-default-class: 'true'
provisioner: rbd.csi.ceph.com
parameters:
  pool: k8s-etc
  clusterID: e285a458-7c95-4187-8129-fbd6c370c537
  imageFeatures: layering
  csi.storage.k8s.io/fstype: ext4
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
  csi.storage.k8s.io/node-stage-secret-namespace: default
  csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: default
  csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
volumeBindingMode: Immediate
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
  - discard
```

Create demo app for expansion:

```yaml
nano testclaim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```yaml
nano testpod.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
#      securityContext:
#        fsGroupChangePolicy: ReadWriteOnceWithFSType
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
          - mountPath: /var/lib/www/html
            name: mypvc
      volumes:
        - name: mypvc
          persistentVolumeClaim:
            claimName: test-claim
```


```bash
kubectl apply -f testclaim.yaml
kubectl apply -f testpod.yaml

kgp
NAME          READY   STATUS    RESTARTS   AGE
nginx-qsw7x   1/1     Running   0          115

kubectl exec -ti nginx-qsw7x -- df -kh|grep var
/dev/rbd7                974M   24K  958M   1% /var/lib/www/html

kgpvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-claim   Bound    pvc-da8019f5-cf01-4ede-8e36-90dd8a79564b   1Gi        RWO            k8s-etc     11m
```

Now we can change the size of the pvc and the operator will extend teh pv fo it automatically:

```bash
kubectl patch pvc test-claim -p '{"spec": {"resources": {"requests": {"storage": "5Gi"}}}}'

kgpvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-claim   Bound    pvc-da8019f5-cf01-4ede-8e36-90dd8a79564b   5Gi        RWO            k8s-etc     11m

kubectl exec -ti nginx-qsw7x -- df -kh|grep var
/dev/rbd7                4.9G   24K  4.9G   1% /var/lib/www/html
```

