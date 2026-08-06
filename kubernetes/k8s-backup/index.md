---
title: Backup your Kubernetes Cluster
url: https://devopstales.github.io/kubernetes/k8s-backup/
date: 2021-03-26
keywords: kubernetes deployment, Helm, Velere, Kyverno, kanister
---


In this post I will show you how you can backup your Kubernetes cluster.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

## Backup Kubernetes objects
To backup kubernetes objects I use Velero (formerly Heptio Ark) for a long time. I thin thi is one of the best solution. Each Velero operation (on-demand backup, scheduled backup, restore) is a custom resource, stored in etcd. A backup opertaion is uploads a tarball of copied Kubernetes objects into cloud object storage. After that calls the cloud provider API to make disk snapshots of persistent volumes, if specified. Optionally you can specify hooks to be executed during the backup. When you create a backup, you can specify a TTL by adding the flag `--ttl <DURATION>`.

### Velero supported providers:

| Provider | Object Store | Volume Snapshotter |
|----------|--------------|--------------------|
| Amazon Web Services (AWS) | AWS S3 | AWS EBS |
| Google Cloud Platform (GCP) | Google Cloud Storage | Google Compute Engine Disks |
| Microsoft Azure | Azure Blob Storage | Azure Managed Disks |
| Portworx | - | Portworx Volume |
| OpenEBS | - | OpenEBS CStor Volume |
| VMware vSphere | - | vSphere Volumes |
| Container Storage Interface (CSI) | - | CSI Volumes |

### Install Velero client

```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.5.3/velero-v1.5.3-linux-amd64.tar.gz
tar zxvf velero-v1.5.3-linux-amd64.tar.gz
sudo cp velero-v1.5.3-linux-amd64/velero /usr/local/bin
```

### Install Velero server component

First you need to create a secret that contains the S3 ccess_key and secret_key. In my case it is called `minio.secret`.

```bash
velero install \
 --provider aws \
 --plugins velero/velero-plugin-for-aws:v1.1.0,velero/velero-plugin-for-csi:v0.1.2  \
 --bucket bucket  \
 --secret-file minio.secret  \
 --use-volume-snapshots=true \
 --backup-location-config region=default,s3ForcePathStyle="true",s3Url=http://minio.mydomain.intra  \
 --snapshot-location-config region=default \
 --features=EnableCSI
```

We need to annotate the snapshot class for Velero to use it to create a snapshots.

```bash
kubectl label VolumeSnapshotClass csi-rbdplugin-snapclass \
velero.io/csi-volumesnapshot-class=true

kubectl label VolumeSnapshotClass csi-cephfsplugin-snapclass \
velero.io/csi-volumesnapshot-class=true
```

### Create Backup
```bash
velero backup create nginx-backup \
--include-namespaces nginx-example --wait

velero backup describe nginx-backup
velero backup logs nginx-backup
velero backup get

velero schedule create nginx-daily --schedule="0 1 * * *" \
--include-namespaces nginx-example

velero schedule get
velero backup get
```

### Automate Backup schedule with kyverno
```bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: autobackup-policy
spec:
  background: false
  rules:
  - name: "add-velero-autobackup-policy"
    match:
        resources:
          kinds:
            - Namespace
          selector:
            matchLabels:
              nirmata.io/auto-backup: enabled
    generate:
        kind: Schedule
        name: "{{request.object.metadata.name}}-auto-schedule"
        namespace: velero
        apiVersion: velero.io/v1
        synchronize: true
        data:
          metadata:
            labels:
              nirmata.io/backup.type: auto
              nirmata.io/namespace: '{{request.object.metadata.name}}'
          spec:
            schedule: 0 1 * * *
            template:
              includedNamespaces:
                - "{{request.object.metadata.name}}"
              snapshotVolumes: false
              storageLocation: default
              ttl: 168h0m0s
              volumeSnapshotLocations:
                - default
```


### Restore test
```bash
kubectl delete ns nginx-example

velero restore create nginx-restore-test --from-backup nginx-backup
velero restore get

kubectl get po -n nginx-example
```

## Backup etcd database

### Etcd Backup with RKE2
With RKE2 the snapshoting of ETCD database is automaticle enabled. You can configure the snapshot interval in the rke2 config like this:

```bash
mkdir -p /etc/rancher/rke2
cat << EOF >  /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
profile: "cis-1.5"
# Make a etcd snapshot every 6 hours
etcd-snapshot-schedule-cron: "0 */6 * * *"
# Keep 56 etcd snapshorts (equals to 2 weeks with 6 a day)
etcd-snapshot-retention: 56
EOF
```

The snapshot directory defaults to `/var/lib/rancher/rke2/server/db/snapshots`

### Restoring RKE2 Cluster from a Snapshot

To restore the cluster from backup, run RKE2 with the `--cluster-reset` option, with the `--cluster-reset-restore-path` also given:

```bash
systemctl stop rke2-server
rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/rancher/rke2/server/db/etcd-old-%date%/
```

**Result:** A message in the logs says that RKE2 can be restarted without the flags. Start RKE2 again and should run successfully and be restored from the specified snapshot.

When rke2 resets the cluster, it creates a file at `/var/lib/rancher/rke2/server/db/etc/reset-file`. If you want to reset the cluster again, you will need to delete this file.

## Backup ETCD with kanister
Kanister is a nother backup tool fro Kubernetes created by Veeam.

### Installing Kanister

```bash
helm repo add kanister https://charts.kanister.io/
helm install --name kanister --namespace kanister kanister/kanister-operator --set image.tag=0.50.0
```

Before taking a backup of the etcd cluster, a Secret needs to be created, containing details about the authentication mechanism used by etcd and another for the S3 bucket. In the case of `kubeadm`, it is likely that etcd will have been deployed using TLS-based authentication.

```bash
kanctl create profile s3compliant --access-key <aws-access-key> \
        --secret-key <aws-secret-key> \
        --bucket <bucket-name> --region <region-name> \
        --namespace kanister

kubectl create secret generic etcd-details \
     --from-literal=cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --from-literal=cert=/etc/kubernetes/pki/etcd/server.crt \
     --from-literal=endpoints=https://127.0.0.1:2379 \
     --from-literal=key=/etc/kubernetes/pki/etcd/server.key \
     --from-literal=etcdns=kube-system \
     --from-literal=labels=component=etcd,tier=control-plane \
     --namespace kanister

kubectl label secret -n kanister etcd-details include=true
kubectl annotate secret -n kanister etcd-details kanister.kasten.io/blueprint='etcd-blueprint'
```

Kanister uses a CRD called `Bluetoprint` to read the backup sequence. There is an example `Bluetoprint` for Etcd backup:

```bash
kubectl --namespace kasten apply -f \
    https://raw.githubusercontent.com/kanisterio/kanister/0.50.0/examples/etcd/etcd-in-cluster/k8s/etcd-incluster-blueprint.yaml
```

Now we can create a backup by createing a CRD called `ActionSet`:

```bash
kubectl create -n kanister -f -
apiVersion: cr.kanister.io/v1alpha1
kind: ActionSet
metadata:
  creationTimestamp: null
  generateName: backup-
  namespace: kanister
spec:
  actions:
  - blueprint: "<blueprint-name>"
    configMaps: {}
    name: backup
    object:
      apiVersion: v1
      group: ""
      kind: ""
      name: "<secret-name>"
      namespace: "<secret-namespace>"
      resource: secrets
    options: {}
    preferredVersion: ""
    profile:
      apiVersion: ""
      group: ""
      kind: ""
      name: "<profile-name>"
      namespace: kanister
      resource: ""
    secrets: {}
EOF

kubectl get actionsets
kubectl describe actionsets -n kanister backup-hnp95
```

### Restore the ETCD cluster

SSH into the node where ETCD is running, most usually it would be Kubernetes master node.

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --data-dir="/var/lib/etcd-from-backup" \
  --initial-cluster="ubuntu-s-4vcpu-8gb-blr1-01-master-1=https://127.0.0.1:2380" \
  --name="ubuntu-s-4vcpu-8gb-blr1-01-master-1" \
  --initial-advertise-peer-urls="https://127.0.0.1:2380" \
  --initial-cluster-token="etcd-cluster-1" \
  snapshot restore /tmp/etcd-backup.db
```

And we will just have to instruct the ETCD that is running to use this new dir instead of the dir that it uses by default. To do that open the static pod manifest for ETCD, that would be `/etc/kubernetes/manifests/etcd.yaml` and 

* change the `data-dir` for the etcd container's command to have `/var/lib/etcd-from-backup`
* add another argument in the command `--initial-cluster-token=etcd-cluster-1` as we have seen in the restore command
* change the volume (named e`tcd-data`) to have new dir `/var/lib/etcd-from-backup`
* change volume mount (named `etcd-data`) to new dir `/var/lib/etcd-from-backup`

once you save this manifest, new ETCD pod will be created with new data dir. Please wait for the ETCD pod to be up and running.

### Restoring ETCD snapshot in case of Multi Node ETCD cluster

If your Kubernetes cluster is setup in such a way that you have more than one memeber of ETCD up and running, you will have to follow almost the same steps that we have
already seen with some minor changes.
So you have one snapshot file from backup and as the [ETCD documentation](https://etcd.io/docs/v3.4.0/op-guide/recovery/) says all the members should restore from the same snapshot. What we would do is choose one leader node that we will be using to restore the backup that we have taken and stop the static pods from all other leader nodes.
To stop the static pods from other leader nodes you will have to move the static pod manifests from the static pod path, which in case of kubeadm is `/etcd/kubernetes/manifests`.
Once you are sure that the containers on the other follower nodes have been stopped, please follow the step that is mentioned previously (`Restore the ETCD cluster`) on all the leader nodes sequentially.

If we take a look into the bellow command that we are actually going to run to restore the snapshot

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --data-dir="/var/lib/etcd-from-backup" \
  --initial-cluster="ubuntu-s-4vcpu-8gb-blr1-01-master-1=https://127.0.0.1:2380" \
  --name="ubuntu-s-4vcpu-8gb-blr1-01-master-1" \
  --initial-advertise-peer-urls="https://127.0.0.1:2380" \
  --initial-cluster-token="etcd-cluster-1" \
  snapshot restore /tmp/etcd-backup.db
```

Make sure to change the of node name for the flag `--initial-cluster` and `--name` because this is going to change based on which leader node you are running the command on.
We want be changing the value of `--initial-cluster-token` because `etcdctl restore` command creates a new member and we want all these new members to have same token, so
that would belong to one cluster and accidently wouldnt join any other one.

To explore more about this we can look into the [Kubernetes documentation](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster).

