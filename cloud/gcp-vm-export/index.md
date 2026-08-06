---
title: Export GCP VM to S3
url: https://devopstales.github.io/cloud/gcp-vm-export/
date: 2019-10-04
keywords: google, GCP, google cloud platform, S3
---


Step by step guide to export virtual machine running in Google cloud computer engine to your S3 bucket.
<!--more-->

```
gcloud compute instances list
NAME        ZONE            MACHINE_TYPE               PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
demo1   europe-west1-b  g1-small                                10.132.0.3                TERMINATED
```

### Create snapshot
```
gcloud compute disks snapshot europe-west1-b/disks/demo1 --storage-locatio europe-west1

gcloud compute snapshots list
NAME              DISK_SIZE_GB  SRC_DISK                        STATUS
demo1-backup  30            europe-west1-b/disks/demo1  READY
```

### Create custom image
```
gcloud compute images create demo1-backup --source-snapshot demo1-backup
Created [https://www.googleapis.com/compute/v1/projects/demo-project-223110/global/images/demo1-backup].
NAME              PROJECT               FAMILY  DEPRECATED  STATUS
demo1-backup  demo-project-223110                      READY
```
### Create S3 storage
```
gsutil mb gs://backup-demo-project-223110/ -l europe-west1
```

### Export to S3 storage
```
gcloud compute images export --destination-uri gs://backup-demo-project-223110/demo1-beckup.tar.gz --image demo1-backup --export-format=vmdk
```

