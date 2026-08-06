---
title: Configure OKD OpenShift 4 Ceph Persisten Storage
url: https://devopstales.github.io/kubernetes/openshift4-ceph-rbd-csi/
date: 2021-12-13
keywords: openshift 4.2, openshift okd4, openshift container platform, rad hat openshift
---



In this Post I will show you how you can create peristent storage on an OpenShift 4 with Ceph RBD CSI Driver.
<!--more-->

{{< content "/filedir/openshift4.html" >}}

| Host | ROLES | OS | IP |
|:------:|:------:|:------:|:------:|
| okd4-ceph1 | master | CentOS 7 | 192.168.1.221 |
| okd4-ceph2 | master | CentOS 7 | 192.168.1.222 |
| okd4-ceph3 | master | CentOS 7 | 192.168.1.223 |
| okd4-ceph4 | OSD | CentOS 7 | 192.168.1.224 |
| okd4-ceph5 | OSD | CentOS 7 | 192.168.1.225 |

First we need a priject where we will install the Ceph Driver:

```bash
oc adm new-project ceph-csi-rbd
oc project ceph-csi-rbd
```

We need the cluster id from the ceph cluster so get it from the ceph cluster

```
ceph -s
  cluster:
    id:     f8b13ea1-2s52-4fe8-bd67-e7ddf259122b
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum okd4-ceph1,okd4-ceph2,okd4-ceph3
    mgr: okd4-ceph3(active), standbys: okd4-ceph2, okd4-ceph1
    mds: cephfs-1/1/1 up  {0=okd4-ceph4=up:active}, 3 up:standby
    osd: 8 osds: 8 up, 8 in
    rgw: 4 daemons active

  data:
    pools:   14 pools, 664 pgs
    objects: 1.72M objects, 6.26TiB
    usage:   16.7TiB used, 26.9TiB / 43.7TiB avail
    pgs:     663 active+clean
             1   active+clean+scrubbing+deep

  io:
    client:   253B/s rd, 2.04MiB/s wr, 0op/s rd, 49op/s wr
```

With the cluster id we can create the value file for the helm chart:

```yaml
nano values.yaml
csiConfig:
  - clusterID: "f8b13ea1-2s52-4fe8-bd67-e7ddf259122b"
    monitors:
      - "192.168.1.221:6789"
      - "192.168.1.222:6789"
      - "192.168.1.223:6789"
```

Create the `SecurityContextConstraints` fot the `ceph-csi-rbd-provisioner`.

```yaml
nano provisioner-scc.yaml
---
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  annotations:
    kubernetes.io/description: ceph-csi-rbd-provisioner scc is used for ceph-csi-rbd-provisioner
  name: ceph-csi-rbd-provisioner
allowHostDirVolumePlugin: true
allowHostIPC: false
allowHostNetwork: true
allowHostPID: true
allowHostPorts: true
allowPrivilegeEscalation: true
allowPrivilegedContainer: true
allowedCapabilities:
  - 'SYS_ADMIN'
priority: null
readOnlyRootFilesystem: false
requiredDropCapabilities: null
defaultAddCapabilities: null
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
fsGroup:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
volumes:
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'hostPath'
users:
  - system:serviceaccount:ceph-csi-rbd:ceph-csi-rbd-provisioner
groups: []
```

```bash
oc apply -f provisioner-scc.yaml

helm repo add ceph-csi https://ceph.github.io/csi-charts 
helm install --namespace "ceph-csi-rbd" "ceph-csi-rbd" ceph-csi/ceph-csi-rbd -f values.yaml
```

Now we need to configure the athentication for the ceph cluster. First get the ceph cluster admin userKey:

```bash
echo "admin" | base64
YWRtaW4K

ceph auth get-key client.admin|base64
QVFDTDliVmNEb21I32SHoPxXNGhmRkczTFNtcXM0ZW5VaXlTZEE977==
```

```yaml
nano secret.yaml
---
kind: Secret
apiVersion: v1
metadata:
  name: csi-rbd-secret
  namespace: default
data:
  userID: YWRtaW4=
  userKey: QVFDTDliVmNEb21I32SHoPxXNGhmRkczTFNtcXM0ZW5VaXlTZEE977==
type: Opaque
```

```yaml
nano okd4-pool.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-rbd-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: 'true'
provisioner: rbd.csi.ceph.com
parameters:
  pool: okd4-pool
  clusterID: f8b13ea1-2s52-4fe8-bd67-e7ddf259122b
  volumeNamePrefix: okd4-vol-
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


```bash
oc apply -f secret.yaml
oc apply -f okd4-pool.yaml
```

```yaml
nano raw-block-pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: raw-block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 50Mi
  storageClassName: csi-rbd-sc
```

The test with a pvc `oc apply raw-block-pvc.yaml`

### Configuring registry storage

Edit the imageregistry-operator:

```bash
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'
oc edit configs.imageregistry.operator.openshift.io

# from
...
spec:
  storage: {}

# to
spec:
  storage:
    pvc:
      claim:
```

```bash
oc get pod -n openshift-image-registry

NAME                                               READY     STATUS    RESTARTS      AGE
cluster-image-registry-operator-5897c9d897-46rg8   1/1       Running   1 (12h ago)   33h
image-registry-7467dd65f9-vhnvx                    1/1       Pending   0             5m
node-ca-6q6bh                                      1/1       Running   2             9d
node-ca-7hphd                                      1/1       Running   2             9d
node-ca-cbt2x                                      1/1       Running   2             9d
node-ca-cfqf5                                      1/1       Running   2             9d
node-ca-gk5ps                                      1/1       Running   2             9d
node-ca-r5jnx                                      1/1       Running   2             9d
```

### Enable the Image Registry default route

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --type merge -p '{"spec":{"defaultRoute":true}}'
```

