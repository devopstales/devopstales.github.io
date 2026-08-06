---
title: Configuringure OKD OpenShift 4 registry for bare metal
url: https://devopstales.github.io/kubernetes/openshift4-registry/
date: 2023-01-23
keywords: openshift 4.11, openshift okd4, openshift container platform, rad hat openshift, docker registry, rad hat quay
---



In this Post I will show you how you can configure the enbedded rad hat quay docker registry in Openshift.
<!--more-->

{{< content "/filedir/openshift4.html" >}}

On platforms that do not provide shareable object storage, the OpenShift Image Registry Operator bootstraps itself as `Removed`. This allows `openshift-installer` to complete installations on these platform types.

### Changing the image registry’s management state

To start the image registry, you must change the Image Registry Operator configuration’s `managementState` from `Removed` to `Managed`.

```bash
oc project openshift-image-registry
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'
```

### Image registry storage configuration

The Image Registry Operator is not initially available for platforms that do not provide default storage. After installation, you must configure your registry to use storage so that the Registry Operator is made available. I configured ceph storage in a [previous post]().

Edit the registry configuration and add `image-registry-storage` PVC.

```bash
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"pvc":{"claim":"image-registry-storage"}}}}'
```

You must configure storage for the Image Registry Operator. For non-production clusters, you can set the image registry to an empty directory. If you do so, all images are lost if you restart the registry.

```bash
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```

### Image registry RBD storage configuration

To allow the image registry to use block storage types such as RBD or vSphere Virtual Machine Disk (VMDK) during upgrades as a cluster administrator, you can use the Recreate rollout strategy.

> Block storage volumes are supported but not recommended for use with image registry on production clusters. An installation where the registry is configured on block storage is not highly available because the registry cannot have more than one replica.

```bash
oc patch config.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"rolloutStrategy":"Recreate","replicas":1}}'
```

Provision the PV for the block storage device, and create a PVC for that volume. 

```bash
nano pvc.yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: image-registry-storage 
spec:
  accessModes:
  - ReadWriteOnce 
  resources:
    requests:
      storage: 100Gi 
```

```bash
oc apply -f pvc.yaml

oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```

### Image registry S3 storage configuration

To allow the image registry to use S3 storage you need to create a `image-registry-private-configuration-user` secret to provide credentials needed for storage access and management. 

### Exposing OpenShift Container Registry

The first step in setting up an OpenShift Container Registry is to expose the registry through the default or customized route. You can do so by running the following command. 

For S3 on AWS storage, the secret is expected to contain two keys:
* `REGISTRY_STORAGE_S3_ACCESSKEY`
* `REGISTRY_STORAGE_S3_SECRETKEY`

Create an OpenShift Container Platform secret that contains the required keys:

```bash
oc create secret generic image-registry-private-configuration-user --from-literal=REGISTRY_STORAGE_S3_ACCESSKEY=myaccesskey --from-literal=REGISTRY_STORAGE_S3_SECRETKEY=mysecretkey --namespace openshift-image-registry
```

You must configure storage for the Image Registry Operator.

```bash
oc edit configs.imageregistry.operator.openshift.io/cluster
...
  storage:
    s3:
      bucket: <bucket-name>
      region: <region-name>
# is you use self hosted S3 like Minio or Ceph

oc patch configs.imageregistry.operator.openshift.io/cluster --type=merge --patch '{"spec":{"defaultRoute":true}}'
```

> If you use self hosted S3 like Minio or Ceph y need to add the `regionEndpoint` option too. For example:
> ```bash
>  storage:
>    s3:
>      bucket: <bucket-name>
>      region: <region-name>
>      regionEndpoint: http://rook-ceph-rgw-ocs-storagecluster-cephobjectstore.openshift-storage.svc.cluster.local
> ```
