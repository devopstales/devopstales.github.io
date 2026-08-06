---
title: Install OpenEBS for Kubernetes
url: https://devopstales.github.io/kubernetes/k8s-install-openebs/
date: 2020-05-20
keywords: kubernetes deployment, kubernetes vs docker, OpenEBS
---


OpenEBS is an open-source project for container-attached and container-native storage on Kubernetes.
<!--more-->

{{< content "/filedir/kubernetes.html" >}}

On all host we have an unused unpartitioned disk called sdb.
```
lsblk

NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0   80G  0 disk
├─sda1            8:1    0    1G  0 part /boot
└─sda2            8:2    0   79G  0 part
  ├─centos-root 253:0    0   50G  0 lvm  /
  ├─centos-swap 253:1    0    1G  0 lvm
  └─centos-home 253:3    0   28G  0 lvm  /home
sdb               8:16   0  100G  0 disk
```

### Install requirements

OpenEBS use iscsi for persisten volume sharing so we need  `iscsid`.

#### Debian / Ubuntu
```
apt-get install open-iscsi
service open-iscsi enable
service open-iscsi restart

```

#### CentOS
```
yum install iscsi-initiator-utils -y
systemctl enable iscsid
systemctl start iscsid
```

### Deploy OpenEBS with helm

OpenEBS is in the stable repo so we didn't need to add a separate helm repository for installing it. In my case I will use teh `openebs-system` namespace for the install.

```
kubectl create ns openebs-system
helm upgrade --install openebs stable/openebs --version 1.7.0 --namespace=openebs-system
```

Wait for all the  pods are started.
```
kubectl get pods -n openebs-system

NAME                                           READY     STATUS    RESTARTS   AGE
openebs-admission-server-7b4859ccd5-bz4zt      1/1       Running   0          14m
openebs-apiserver-556ffff45c-nk9x9             1/1       Running   5          15m
openebs-localpv-provisioner-76b466d4b8-5tj4w   1/1       Running   0          15m
openebs-ndm-f6cqz                              1/1       Running   0          15m
openebs-ndm-operator-5f6c5497d7-chf6t          1/1       Running   1          15m
openebs-ndm-qrmp9                              1/1       Running   0          15m
openebs-ndm-stgml                              1/1       Running   0          15m
openebs-provisioner-c9c7f9ff8-hn4bl            1/1       Running   0          15m
openebs-snapshot-operator-6578d74b7-2wc97      2/2       Running   0          15m
```

Now verify if OpenEBS is installed successfully.
```
kubectl get blockdevice -n openebs-system

NAME                                           NODENAME    SIZE         CLAIMSTATE   STATUS    AGE
blockdevice-0c4e03f9e39a4092108215f19eca9da8   k8s-node1   1048576000   Unclaimed    Active    16m
blockdevice-1aaa1142a7b9c65dfa32dec88fe1749b   k8s-node2   1048576000   Unclaimed    Active    16m
blockdevice-5f728d1068c72337609fc1f88855b9bb   k8s-node3   1048576000   Unclaimed    Active    16m
```

```
kubectl describe blockdevice blockdevice-0c4e03f9e39a4092108215f19eca9da8
...
  Devlinks:
    Kind:  by-id
    Links:
      /dev/disk/by-id/ata-VBOX_HARDDISK_VBd4679835-eb798f2c
      /dev/disk/by-id/lvm-pv-uuid-PWnLFv-b0jS-7CLZ-Cmym-0dia-RQkI-w0Hkam
    Kind:  by-path
    Links:
      /dev/disk/by-path/pci-0000:00:01.1-ata-2.0
  Filesystem:
    Fs Type:  LVM2_member
  Node Attributes:
    Node Name:  k8s-node1
  Partitioned:  No
  Path:         /dev/sdb
...
```

#### Verify StorageClasses:
```
kubectl get sc

NAME                        PROVISIONER                                                AGE
openebs-device              openebs.io/local                                           64s
openebs-hostpath            openebs.io/local                                           64s
openebs-jiva-default        openebs.io/provisioner-iscsi                               64s
openebs-snapshot-promoter   volumesnapshot.external-storage.k8s.io/snapshot-promoter   64s
```

### Storage engines
OpenEBS offers three storage engines:
* Jiva
* cStor
* LocalPV

Jiva is a light weight storage engine that is recommended to use for low capacity workloads. It is actually based on the same technology that powers Longhorn.

```
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: demo-vol1-claim
spec:
  storageClassName: openebs-jiva-default
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4G
```

cStor requires raw disks. The snapshot and storage management features of the other cStor engine are more advanced than Jiva. For provisioning a cStor Volume, it requires a cStor Storage Pool and a StorageClass. cStor provides iSCSI targets, which are appropriate for RWO (ReadWriteOnce) access mode and is suitable for all types of databases. cStor supports thin provisioning by default.

```
cat stor-pool1-config.yaml
---
apiVersion: openebs.io/v1alpha1
kind: StoragePoolClaim
metadata:
  name: cstor-disk-pool
  annotations:
    cas.openebs.io/config: |
      - name: PoolResourceRequests
        value: |-
            memory: 500Mb
      - name: PoolResourceLimits
        value: |-
            memory: 500Mb
spec:
  name: cstor-disk-pool
  type: disk
  poolSpec:
    poolType: striped
  blockDevices:
    blockDeviceList:
    - blockdevice-0c4e03f9e39a4092108215f19eca9da8
    - blockdevice-1aaa1142a7b9c65dfa32dec88fe1749b
    - blockdevice-5f728d1068c72337609fc1f88855b9bb
```

```
kubectl apply -f stor-pool1-config.yaml

kubectl get spc

NAME              AGE
cstor-disk-pool   20s

kubectl get csp

NAME                   ALLOCATED   FREE      CAPACITY   STATUS    TYPE      AGE
cstor-disk-pool-cxm8   294K        100G      100G       Healthy   striped   27m
cstor-disk-pool-r1hl   270K        100G      100G       Healthy   striped   27m
cstor-disk-pool-t05z   92K         100G      100G       Healthy   striped   27m
```

```
cat openebs-sc-rep3.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-cstore-default
  annotations:
    openebs.io/cas-type: cstor
    cas.openebs.io/config: |
      - name: StoragePoolClaim
        value: "cstor-disk-pool"
      - name: ReplicaCount
        value: "3"
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: openebs.io/provisioner-iscsi
```

```
cat test-cs-pcv.yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-cs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3G
```

```
kubectl apply -f openebs-sc-rep3.yaml
kubectl apply -f test-cs-pcv.yaml
```

Local PV is based on Kubernetes local persistent volumes but it has a dynamic provisioner. It can store data either in a directory, or use disks; in the first case the hostpath can be shared by multiple persistent volumes, while when using disks each persistent volume requires a separate device. Local PV offers extremely high performance close to what you get by reading from and writing to the disk directly, but it doesn’t offer features such as replication, which are built in Jiva and cStor.

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    cas.openebs.io/config: |
      - name: StorageType
        value: "hostpath"
      - name: BasePath
        value: "/mnt/openebs"
    openebs.io/cas-type: local
  name: openebs-hostpath-mount
provisioner: openebs.io/local
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

Sources:
---
* https://vitobotta.com/2019/07/03/openebs-tips/

