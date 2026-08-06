---
title: How to fixing filesystem corruption on a Kubernetes Ceph RBD PersistentVolume
url: https://devopstales.github.io/kubernetes/openshift-rbd-fsck/
date: 2020-10-10
keywords: kubernetes deployment, Ceph RBD, PersistentVolume, openshift
---


In this tutorial I will show you how to fix a corruptid filesystem on Ceph RBD PersistentVolume uyed by Kubernetes.

<!--more-->

```
oc describe po gitlab-ce-1-wl9wf
...
Events:
  Type     Reason                  Age               From                                           Message
  ----     ------                  ----              ----                                           -------
  Normal   Scheduled               27s               default-scheduler                              Successfully assigned gitlab-prod/gitlab-ce-1-j7lph to k8sw09
  Normal   SuccessfulAttachVolume  27s               attachdetach-controller                        AttachVolume.Attach succeeded for volume "pvc-e27f498e-85cf-11e9-af1a-66934f1af826"
  Warning  FailedMount             2s (x6 over 19s)  kubelet, k8sw09  MountVolume.MountDevice failed for volume "pvc-e27f498e-85cf-11e9-af1a-66934f1af826" : rbd: failed to mount device /dev/rbd3 at /var/lib/origin/openshift.local.volumes/plugins/kubernetes.io/rbd/mounts/k8s-rbd-image-kubernetes-dynamic-pvc-e3042618-85cf-11e9-8762-aa9884bf6706 (fstype: ), error 'fsck' found errors on device /dev/rbd3 but could not correct them: fsck from util-linux 2.23.2
/dev/rbd3: Superblock needs_recovery flag is clear, but journal has data.
/dev/rbd3: Run journal anyway

/dev/rbd3: UNEXPECTED INCONSISTENCY; RUN fsck MANUALLY.
  (i.e., without -a or -p options)
```

Check the log on the worker. In my case this is k8sw09.
```
journalctl -u kubelet


jan 08 15:44:58 k8sw09 origin-node[14927]: I0108 15:44:58.251201   14927 reconciler.go:252] operationExecutor.MountVolume started for volume "pvc-e27f498e-85cf-11e9-af1a-66934f1af826" (UniqueName: "kubernetes.io/rbd/k8s-rbd:kubernetes-dynamic-pvc-e3042618-85cf-11e9-8762-aa9884bf6706") pod "gitlab-ce-1-j7lph" (UID: "69151c2c-3223-11ea-9bcf-aa9884bf6706")
jan 08 15:44:58 k8sw09 origin-node[14927]: I0108 15:44:58.251299   14927 operation_generator.go:489] MountVolume.WaitForAttach entering for volume "pvc-e27f498e-85cf-11e9-af1a-66934f1af826" (UniqueName: "kubernetes.io/rbd/k8s-rbd:kubernetes-dynamic-pvc-e3042618-85cf-11e9-8762-aa9884bf6706") pod "gitlab-ce-1-j7lph" (UID: "69151c2c-3223-11ea-9bcf-aa9884bf6706") DevicePath ""
jan 08 15:44:58 k8sw09 origin-node[14927]: I0108 15:44:58.451965   14927 operation_generator.go:498] MountVolume.WaitForAttach succeeded for volume "pvc-e27f498e-85cf-11e9-af1a-66934f1af826" (UniqueName: "kubernetes.io/rbd/k8s-rbd:kubernetes-dynamic-pvc-e3042618-85cf-11e9-8762-aa9884bf6706") pod "gitlab-ce-1-j7lph" (UID: "69151c2c-3223-11ea-9bcf-aa9884bf6706") DevicePath "/dev/rbd3"
jan 08 15:44:58 k8sw09 origin-node[14927]: E0108 15:44:58.498052   14927 nestedpendingoperations.go:267] Operation for "\"kubernetes.io/rbd/k8s-rbd:kubernetes-dynamic-pvc-e3042618-85cf-11e9-8762-aa9884bf6706"" failed. No retries permitted until 2020-01-08 15:47:00.498014981 +0100 CET m=+619493.508747496 (durationBeforeRetry 2m2s). Error: "MountVolume.MountDevice failed for volume \"pvc-e27f498e-85cf-11e9-af1a-66934f1af826\" (UniqueName: \"kubernetes.io/rbd/k8s-rbd:kubernetes-dynamic-pvc-e3042618-85cf-11e9-8762-aa9884bf6706\") pod \"gitlab-ce-1-j7lph\" (UID: \"69151c2c-3223-11ea-9bcf-aa9884bf6706\") : rbd: failed to mount device /dev/rbd3 at /var/lib/origin/openshift.local.volumes/plugins/kubernetes.io/rbd/mounts/k8s-rbd-image-kubernetes-dynamic-pvc-e3042618-85cf-11e9-8762-aa9884bf6706 (fstype: ), error 'fsck' found errors on device /dev/rbd3 but could not correct them: fsck from util-linux 2.23.2\n/dev/rbd3: Superblock needs_recovery flag is clear, but journal has data.\n/dev/rbd3: Run journal anyway\n\n/dev/rbd3: UNEXPECTED INCONSISTENCY; RUN fsck MANUALLY.\n\t(i.e., without -a or -p options)\n."
```

We can see the problem is with `/dev/rbd3`. First check thi is the block device user for `pvc-e3042618-85cf-11e9-8762-aa9884bf6706` PersistenVolume.

```
sudo rbd showmapped | grep pvc-e3042618-85cf-11e9-8762-aa9884bf6706
3  k8s-rbd kubernetes-dynamic-pvc-e3042618-85cf-11e9-8762-aa9884bf6706 -    /dev/rbd3
```
So let's try to use `fsck` on this disk.
```
sudo rbd unmap /dev/rbd3


sudo fsck -fv /dev/rbd3
fsck from util-linux 2.27.1
e2fsck 1.42.13 (17-May-2015)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Unattached inode 303
Connect to /lost+found<y>? yes
Inode 303 ref count is 2, should be 1.  Fix<y>? yes
Pass 5: Checking group summary information
Block bitmap differences:  -(71680--73727) -(94208--95231)
Fix<y>? yes

/dev/rbd3: ***** FILE SYSTEM WAS MODIFIED *****

         326 inodes used (0.50%, out of 65536)
          35 non-contiguous files (10.7%)
           0 non-contiguous directories (0.0%)
             # of inodes with ind/dind/tind blocks: 0/0/0
             Extent depth histogram: 311/7
       63642 blocks used (24.28%, out of 262144)
           0 bad blocks
           1 large file

         308 regular files
           9 directories
           0 character device files
           0 block device files
           0 fifos
           1 link
           0 symbolic links (0 fast symbolic links)
           0 sockets
------------
         317 files
```

Then our pod is running again!
```
oc get po
NAME                        READY     STATUS    RESTARTS   AGE
gitlab-ce-1-j7lph           1/1       Running   0          28m
```

