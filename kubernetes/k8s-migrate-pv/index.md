---
title: How to Migrate Persistent Volumes on Kubernetes Easily?
url: https://devopstales.github.io/kubernetes/k8s-migrate-pv/
date: 2023-05-31
keywords: rook, ceph, kubernetes, deployment, pv-migrate, pv, pvc, migrate, storage migration
---


In this post I will show you how can you use migrate your data from one PV to another.

<!--more-->

Have you faced a situation where you need to migrate your data from one PV to another. For example your your DB grew over time, and you need more space for it, but you cannot resize the PVC because it doesn't support volume expansion. Or you need to migrate data between StotageClasses or Clusters. I have, and find a tool called [pv-migrate](https://github.com/utkuozdemir/pv-migrate) that supports this migrations.

For the example I will migrate data between StotageClasses. I have two StotageClasses `cis-rbd-sc` and `csi-cephfs-sc` and I will migrate data from `cis-rbd-sc` to `cis-cephfs-sc`.

### Installing PV-Migrate

First we need to install the `pv-migrate` tool. The easiest way is installing as a kubectl plugin by `krew`:

```bash
kubectl krew install pv-migrate

kubectl pv-migrate --help
```

`pv-migrate` can migrate your data by tree different methods:

* *lbsvc*: Load Balancer Service, this will run rsync+ssh over a Kubernetes Service type LoadBalancer. This is the method you want to use if you're migrating PVC from different Kubernetes clusters.
* *mnt2*: Mounts both PVCs in a single pod and runs a regular rsync. This is only usable if source and destination PVCs are in the same namespace.
* *svc*: Service, Runs rsync+ssh in a Kubernetes Service (ClusteRIP). Only applicable when the source and destination PVCs are in the same Kubernetes cluster.

In my sase it will use the `mnt2` method but you didn't need to configure this because it automatically detect the best method.

### How to Migrate Persistent Volumes

First you need to scale down your application that use the pvc:

```bash
kubectl scale deployment test-app --replicas=0
```

Then we need to create a backup of the `pv` and `pvc` objects:

```bash
kubectl get pv test-app-pv -o yaml > test-app-pv.yaml

# then find the pv belong to it and backup that too:

kg pv pvc-4dce1b52-818e-4e3e-b968-c2fe4e9de7d6 -o yaml > pvc-4dce1b52-818e-4e3e-b968-c2fe4e9de7d6.yaml
```

Now we can edit the existing pvc to change the storage class:

```bash
cp test-app-pv.yaml test-app-pv_new.yaml
```

Remove the unnecessary parts and change the storage class.

```yaml
nano test-app-pv_new.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app: test-app-pv
  name: test-app-pv_new
  namespace: pv-test
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: csi-cephfs-sc
```

Apply the new `pvc`:

```bash
kubectl apply -f test-app-pv_new.yaml

kubectl get pvc test-app-pv test-app-pv_new
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
test-app-pv       Bound    pvc-4dce1b52-818e-4e3e-b968-c2fe4e9de7d6   2Gi        RWO            csi-rbd-sc      54m
test-app-pv_new   Bound    pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39   2Gi        RWO            csi-cephfs-sc   34m
```

Now we can use the `pv-migrate` to copy the data:

```bash
kubectl pv-migrate migrate \
  --ignore-mounted \
  test-app-pv test-app-pv_new
🚀  Starting migration
💡  PVC grafana-two is mounted to node minikube, ignoring...
💡  PVC grafana is mounted to node minikube, ignoring...
💭  Will attempt 3 strategies: mnt2,svc,lbsvc
🚁  Attempting strategy: mnt2
🔑  Generating SSH key pair
🔑  Creating secret for the public key
🚀  Creating sshd pod
⏳  Waiting for the sshd pod to start running
🚀  Sshd pod started
🔑  Creating secret for the private key
🔗  Connecting to the rsync server
📂  Copying data... 100% |█████████████████████████████████████████████████████████████████████████| ()
🧹  Cleaning up
✨  Cleanup successful
✅  Migration succeeded
```

We will delete the `pvc`s so we need to change the reclaim policy:

```bash
kubectl patch pv pvc-4dce1b52-818e-4e3e-b968-c2fe4e9de7d6 -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
kubectl patch pv pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39 -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

kubectl get pv pvc-4dce1b52-818e-4e3e-b968-c2fe4e9de7d6 pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                    STORAGECLASS    REASON AGE
pvc-4dce1b52-818e-4e3e-b968-c2fe4e9de7d6   2Gi        RWO            Retain           Bound       pv-test/test-app-pv       csi-rbd-sc            54m
pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39   2Gi        RWO            Retain           Bound       pv-test/test-app-pv_new   csi-cephfs-sc         34m
```

Now we can delete the `pvc`s:

```bash
kubectl delete -f test-app-pv.yaml
kubectl delete -f test-app-pv_new.yaml

kubectl get pv pvc-4dce1b52-818e-4e3e-b968-c2fe4e9de7d6 pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS         CLAIM                 STORAGECLASS    REASON   AGE
pvc-4dce1b52-818e-4e3e-b968-c2fe4e9de7d6   2Gi        RWO            Retain           Released       pv-test/test-app-pv   csi-rbd-sc            54m
pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39   2Gi        RWO            Retain           Released       pv-test/test-app-pv_new   csi-cephfs-sc            34m
```

Now we need to patch path the `pv`s to reuse:

```bash
kubectl patch pv pvc-4dce1b52-818e-4e3e-b968-c2fe4e9de7d6 -p '{"spec":{"claimRef": null}}'
kubectl patch pv pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39 -p '{"spec":{"claimRef": null}}'

kubectl get pv pvc-4dce1b52-818e-4e3e-b968-c2fe4e9de7d6 pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS      REASON   AGE
pvc-4dce1b52-818e-4e3e-b968-c2fe4e9de7d6   2Gi        RWO            Retain           Available           csi-rbd-sc                 59d
pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39   2Gi        RWO            Retain           Available           csi-cephfs-sc              39d
```

Create the new `test-app-pv` with bound to `pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39`:

```yaml
nano test-app-pv_mod.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app: test-app-pv
  name: test-app-pv
  namespace: pv-test
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: csi-cephfs-sc
  volumeMode: Filesystem
  volumeName: pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39
```

```bash
kubectl apply -f test-app-pv_mod.yaml

kubectl get pvc test-app-pv
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
test-app-pv       Bound    pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39   2Gi        RWO            csi-cephfs-sc   54m

kubectl get pv pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                    STORAGECLASS    REASON AGE
pvc-2aef0615-5e1e-4f66-a14d-cdda326bff39   2Gi        RWO            Retain           Bound       pv-test/test-app-pv   csi-cephfs-sc         34m
```

