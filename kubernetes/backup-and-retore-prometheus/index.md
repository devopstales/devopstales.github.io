---
title: How to Backup and Restore Prometheus
url: https://devopstales.github.io/kubernetes/backup-and-retore-prometheus/
date: 2026-03-18
keywords: Prometheus, Backup, Restore, prometheus-operator, kube-prometheus-stack, Velero, Kubernetes backup
---


Backing up Prometheus is critical for preserving your monitoring history and metrics data. This updated guide for 2026 covers modern backup strategies using the Prometheus Admin API, Velero, and volume snapshots in Kubernetes environments with prometheus-operator.

<!--more-->

## Backup Strategies Overview

| Method | Pros | Cons | Best For |
|--------|------|------|----------|
| **Admin API Snapshot** | Built-in, no extra tools | Manual, requires API access | Quick backups, small deployments |
| **Velero** | Automated, cluster-wide | Additional complexity | Production clusters |
| **Volume Snapshots** | Fast, storage-level | Storage-class dependent | Large deployments |
| **Thanos/Cortex** | Long-term storage, HA | Complex setup | Enterprise deployments |

## Prerequisites

- **Kubernetes cluster** (1.28+)
- **prometheus-operator** or **kube-prometheus-stack** installed
- **kubectl** configured
- **Storage class** with snapshot support (for volume-based backups)
- **Velero** (optional, for automated backups)

## Method 1: Prometheus Admin API Snapshot

### Enable Admin API

The Admin API is disabled by default. Enable it in your Prometheus custom resource:

```bash
# For prometheus-operator
kubectl -n monitoring patch prometheus prometheus \
  --type merge --patch '{"spec":{"enableAdminAPI":true}}'

# For kube-prometheus-stack (Helm)
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --set prometheus.prometheusSpec.enableAdminAPI=true
```

### Verify Admin API is Enabled

```bash
# Port forward to Prometheus
kubectl -n monitoring port-forward svc/prometheus-operated 9090

# Test Admin API
curl http://localhost:9090/api/v2/status
```

### Create a Snapshot

```bash
# Create snapshot via API
curl -XPOST http://localhost:9090/api/v2/admin/tsdb/snapshot

# Response:
# {"status":"success","data":{"name":"20260315T123913Z-6e661e92759805f5"}}
```

**With retention (keeps only recent data):**

```bash
curl -XPOST http://localhost:9090/api/v2/admin/tsdb/snapshot?skip_head=true
```

### Locate and Copy Snapshot Data

```bash
# Find the snapshot directory
kubectl -n monitoring exec -it prometheus-prometheus-operated-prometheus-0 \
  -c prometheus -- ls -la /prometheus/snapshots/

# Copy snapshot to local machine
kubectl -n monitoring cp \
  prometheus-prometheus-operated-prometheus-0:/prometheus/snapshots/20260315T123913Z-6e661e92759805f5 \
  ./prometheus-backup-snapshot -c prometheus
```

### Automate with Script

```bash
#!/bin/bash
# prometheus-backup.sh

NAMESPACE="monitoring"
PROMETHEUS_SVC="prometheus-operated"
BACKUP_DIR="./prometheus-backups"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p $BACKUP_DIR

# Port forward in background
kubectl -n $NAMESPACE port-forward svc/$PROMETHEUS_SVC 9090 &
PF_PID=$!

# Wait for port forward
sleep 3

# Create snapshot
SNAPSHOT_NAME=$(curl -s -XPOST http://localhost:9090/api/v2/admin/tsdb/snapshot | \
  jq -r '.data.name')

echo "Created snapshot: $SNAPSHOT_NAME"

# Copy snapshot
kubectl -n $NAMESPACE cp \
  prometheus-prometheus-operated-prometheus-0:/prometheus/snapshots/$SNAPSHOT_NAME \
  $BACKUP_DIR/$DATE-$SNAPSHOT_NAME -c prometheus

# Cleanup
kill $PF_PID

echo "Backup completed: $BACKUP_DIR/$DATE-$SNAPSHOT_NAME"
```

## Method 2: Velero Backup

### Install Velero

```bash
# Add Velero Helm repository
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update

# Install Velero (example with AWS S3)
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --set configuration.provider=aws \
  --set configuration.backupStorageLocation.bucket=velero-backups \
  --set configuration.backupStorageLocation.config.region=us-east-1 \
  --set credentials.secretContents.credentials="[default]
aws_access_key_id=YOUR_ACCESS_KEY
aws_secret_access_key=YOUR_SECRET_KEY" \
  --set snapshotsEnabled=true
```

### Create Backup Schedule for Prometheus

```bash
# Create backup for Prometheus namespace
velero backup create prometheus-backup --include-namespaces monitoring

# Create scheduled backup (daily at 2 AM)
velero schedule create prometheus-daily \
  --schedule="0 2 * * *" \
  --include-namespaces monitoring \
  --ttl 72h

# Verify backup
velero backup describe prometheus-backup
velero backup logs prometheus-backup
```

### Restore from Velero Backup

```bash
# List available backups
velero backup get

# Restore Prometheus namespace
velero restore create --from-backup prometheus-backup

# Restore with namespace mapping
velero restore create \
  --from-backup prometheus-backup \
  --namespace-mappings monitoring:monitoring-restored
```

## Method 3: Volume Snapshot (CSI)

### Prerequisites

- **CSI driver** with snapshot support
- **VolumeSnapshotClass** configured

```bash
# Check for VolumeSnapshotClass
kubectl get volumesnapshotclass

# Create if not exists (example for AWS EBS)
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete
EOF
```

### Create Volume Snapshot

```bash
# Find Prometheus PVC
kubectl -n monitoring get pvc -l app.kubernetes.io/name=prometheus

# Create VolumeSnapshot
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: prometheus-snapshot-$(date +%Y%m%d-%H%M%S)
  namespace: monitoring
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: prometheus-prometheus-operated-prometheus-db-prometheus-prometheus-operated-prometheus-0
EOF
```

### Automate with CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: prometheus-snapshot
  namespace: monitoring
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: prometheus-snapshot-sa
          containers:
          - name: snapshot
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              SNAPSHOT_NAME="prometheus-snapshot-$(date +%Y%m%d-%H%M%S)"
              kubectl create volumesnapshot $SNAPSHOT_NAME \
                --namespace monitoring \
                --volume-snapshot-class ebs-snapshot-class \
                --source persistentvolumeclaim/prometheus-pvc
              # Cleanup old snapshots (keep last 7)
              kubectl get volumesnapshot -n monitoring \
                --sort-by=.metadata.creationTimestamp \
                -o name | head -n -7 | xargs -r kubectl delete -n monitoring
          restartPolicy: OnFailure
```

## Restore Prometheus Data

### Method 1: Restore from Admin API Snapshot

```bash
# Stop Prometheus (scale down statefulset)
kubectl -n monitoring scale statefulset prometheus-prometheus-operated-prometheus --replicas=0

# Delete existing data
kubectl -n monitoring exec -it prometheus-prometheus-operated-prometheus-0 \
  -c prometheus -- rm -rf /prometheus/*

# Copy backup data
kubectl -n monitoring cp ./prometheus-backup-snapshot \
  prometheus-prometheus-operated-prometheus-0:/prometheus/ -c prometheus

# Restart Prometheus
kubectl -n monitoring scale statefulset prometheus-prometheus-operated-prometheus --replicas=1
```

### Method 2: Restore from Volume Snapshot

```bash
# Create new PVC from snapshot
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-restored-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp2
  dataSource:
    name: prometheus-snapshot-20260315-123456
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  resources:
    requests:
      storage: 100Gi
EOF

# Update Prometheus to use restored PVC
kubectl -n monitoring patch prometheus prometheus \
  --type merge --patch '{"spec":{"storage":{"volumeClaimTemplate":{"spec":{"selector":{"matchLabels":{"app":"prometheus-restored"}}}}}}'
```

### Method 3: Full Namespace Restore with Velero

```bash
# Delete existing Prometheus resources
kubectl -n monitoring delete statefulset prometheus-prometheus-operated-prometheus
kubectl -n monitoring delete pvc -l app.kubernetes.io/name=prometheus

# Restore from Velero backup
velero restore create prometheus-restore \
  --from-backup prometheus-backup \
  --include-namespaces monitoring
```

## Long-Term Storage with Thanos

For production environments, consider Thanos for long-term retention:

### Thanos Sidecar Configuration

```yaml
# Add to prometheus-operator Prometheus CR
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  thanos:
    baseImage: quay.io/thanos/thanos
    version: v0.34.0
    objectStorageConfig:
      key: thanos.yaml
      name: thanos-objstore-config
```

### Object Storage Config

```yaml
# thanos-objstore-config Secret
apiVersion: v1
kind: Secret
metadata:
  name: thanos-objstore-config
  namespace: monitoring
type: Opaque
stringData:
  thanos.yaml: |
    type: S3
    config:
      bucket: prometheus-metrics
      endpoint: s3.amazonaws.com
      region: us-east-1
      access_key: YOUR_ACCESS_KEY
      secret_key: YOUR_SECRET_KEY
```

## Backup Best Practices

### Retention Policies

```yaml
# Prometheus retention settings
spec:
  retention: 15d          # Keep 15 days of data
  retentionSize: 50GB     # Or 50GB limit
  resources:
    requests:
      memory: 2Gi
      cpu: 500m
    limits:
      memory: 4Gi
      cpu: 1000m
```

### Backup Schedule Recommendations

| Environment | Frequency | Retention |
|-------------|-----------|-----------|
| **Development** | Weekly | 7 days |
| **Staging** | Daily | 14 days |
| **Production** | Hourly (incremental) + Daily | 90 days |

### Monitoring Backups

```yaml
# PrometheusRule for backup monitoring
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: backup-alerts
  namespace: monitoring
spec:
  groups:
  - name: backup.rules
    rules:
    - alert: PrometheusBackupFailed
      expr: increase(prometheus_tsdb_compactions_failed_total[1h]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Prometheus backup failed"
        description: "Prometheus compaction failed on {{ $labels.instance }}"
    
    - alert: PrometheusBackupOld
      expr: (time() - prometheus_tsdb_data_replay_duration_seconds) > 86400
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "Prometheus backup is stale"
        description: "Last successful backup was more than 24 hours ago"
```

## Troubleshooting

### Admin API Not Responding

```bash
# Check if Admin API is enabled
kubectl -n monitoring exec -it prometheus-prometheus-operated-prometheus-0 \
  -c prometheus -- ps aux | grep enable-admin-api

# Check Prometheus logs
kubectl -n monitoring logs prometheus-prometheus-operated-prometheus-0 -c prometheus \
  | grep -i admin
```

### Snapshot Fails

```bash
# Check disk space
kubectl -n monitoring exec -it prometheus-prometheus-operated-prometheus-0 \
  -c prometheus -- df -h /prometheus

# Check Prometheus TSDB status
curl http://localhost:9090/api/v2/status/tsdb
```

### Restore Issues

```bash
# Verify PVC is bound
kubectl -n monitoring get pvc

# Check Prometheus pod events
kubectl -n monitoring describe pod prometheus-prometheus-operated-prometheus-0

# Verify data directory permissions
kubectl -n monitoring exec -it prometheus-prometheus-operated-prometheus-0 \
  -c prometheus -- ls -la /prometheus/
```

## Migration to New Prometheus Instance

```bash
# Export configuration
kubectl -n monitoring get prometheus prometheus -o yaml > prometheus-config.yaml

# Remove sensitive fields
kubectl -n monitoring get secret prometheus-prometheus-operated-prometheus \
  -o yaml > prometheus-secret.yaml

# Apply to new cluster
kubectl apply -f prometheus-config.yaml
kubectl apply -f prometheus-secret.yaml

# Restore data from backup
# (Use one of the restore methods above)
```

