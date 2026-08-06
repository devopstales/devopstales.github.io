---
title: trivy-operator 2.5: Patch release for Admisssion controller
url: https://devopstales.github.io/kubernetes/trivy-operator-2.5/
date: 2022-10-15
keywords: Kubernetes, trivy-operator, trivy, operator, Image scan, image security
---


Today I am happy to announce the release of trivy-operator 2.5. This blog post focuses on the functionality provided by the trivy-operator 2.5 release. 

<!--more-->

### What is trivy-operator?

Trivy-operator is a Kubernetes Operator based on the open-source container vulnerability scanner Trivy. The goal of this project is to provide a vulnerability scanner that continuously scans containers deployed in a Kubernetes cluster. [Built with Kubernetes Operator Pythonic Framework (Kopf)](https://github.com/nolar/kopf) There are a few solution for checking the images when you deploy them to the Kubernetes cluster, but fighting against vulnerabilities is a day to day task. Check once is not enough when every day is a new das for frats. That is why I created trivy-operator so you can create scheduled image scans on your running pods.

## What is new?

With the release of trivy-operator 2.5 ther is the fallowin new features:

* air-gap install
* Kubernetes CIS Benchmark with kube-bench-scnner
* defectdojo integration
* use insecure registry
* add new dashboard

### Air-Gapped Install

To run trivy-operator in an air-gapped environment you need to provide the security database for trivy. You can do that by uploading tha database to an OCI compatible registry.

```bash
oras pull ghcr.io/aquasecurity/trivy-db:2

oras push docker.mydomain.intra/trivy-db:2 \
db.tar.gz:application/vnd.aquasec.trivy.db.layer.v1.tar+gzip

curl -X GET https://docker.mydomain.intra/v2/_catalog
{"repositories":["nginx","trivy-db"]}

curl -X GET https://docker.mydomain.intra/v2/trivy-db/tags/list
{"name":"trivy-db","tags":["2"]}
```

In the helm chart you need to specify the url of your OCI registry with the `db_repository` option.

```bash
# Don't try to download trivy db, run in air-gapped env:
offline:
  enabled: true
  db_repository: docker.mydomain.intra/trivy-db
```

### Kubernetes CIS Benchmark

CIS Benchmark best practices are an important first step to securing Kubernetes in production by hardening Kubernetes environments. Trivy-operator use kube-bench to scan the kubernetes cluster and create CIS Benchmark reports. To enable the CIS Benchmark scanning function you need to create a ClusterScanner.

The following example object is configured to:

* run the vulnerability scan every hour (`crontab: '00 * * * *'`)
* use the `cis-1.23` scan profile
* enable integration to defectdojo


```yaml
apiVersion: trivy-operator.devopstales.io/v1
kind: ClusterScanner
metadata:
  name: main-config
spec:
  crontab: "00 * * * *"
  scanProfileName: "cis-1.23"
  integrations:
    defectdojo:
      host: "http://defectdojo.rancher-desktop.intra"
      api_key: "3880d84590915e5c96cec075444f22285ff3659c"
      k8s-cluster-name: "eks-prod"
```

The following list show the ClusterScanner objects listed by the kubectl cli:

```bash
kubectl get cs-scan
NAME          CLUSTERSCANPROFILE   CRONTAB
main-config   cis-1.23             00 * * * *
```

### Enable DefectDojo integration for trivy-operator

To enable the DefectDojo integration for trivy-operator you need to enable it in the `NamespaceScanner` object:

```yaml
  integrations:
    policyreport: True
    defectdojo:
      host: "https://defectdojo.mydomain.intra"
      api_key: "xyz456ucdssd67sd67dsg"
```

## Usage

To ease deployment I created a helm chart for trivy-operator.

```bash
helm repo add devopstales https://devopstales.github.io/helm-charts
helm repo update
```

Create a value file for deploy:

```yaml
cat <<'EOF'> values.yaml
image:
  repository: devopstales/trivy-operator
  pullPolicy: Always
  tag: "2.3"

imagePullSecrets: []
podSecurityContext:
  fsGroup: 10001
  fsGroupChangePolicy: "OnRootMismatch"

serviceAccount:
  create: true
  annotations: {}
  name: "trivy-operator"

monitoring:
  port: "9115"

serviceMonitor:
  enabled: false
  namespace: "kube-system"

storage:
  enabled: true
  size: 1Gi

NamespaceScanner:
  crontab: "*/5 * * * *"
  namespaceSelector: "trivy-scan"

registryAuth:
  enabled: false
  registry:
  - name: docker.io
    user: "user"
    password: "password"

githubToken:
  enabled: false
  token: ""
EOF
```

When the trivy in the container want to scan an image first download the vulnerability database from github. If you test many images you need a  `githubToken` overcome the github rate limit and dockerhub username and password for overcome the dockerhub rate limit. If your store you images in a private repository you need to add an username and password for authentication.

The following tables lists configurable parameters of the trivy-operator chart and their default values.

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| TimeZone | string | `"UTC"` | Time Zone in container |
| admissionController.enabled | bool | `false` | enable adission controller |
| affinity | object | `{}` | Set the affinity for the pod. |
| cache.enabled | bool | `false` | enable redis cache |
| clusterScanner.crontab | string | `"*/1 * * * *"` | crontab for scheduled scan |
| clusterScanner.enabled | bool | `false` | enable clusterScanner cr creation |
| clusterScanner.integrations | object | `{}` | configure defectdojo integration |
| clusterScanner.scanProfileName | string | `"cis-1.23"` | kube-hunter scan profile |
| githubToken.enabled | bool | `false` | enable github authentiation token |
| githubToken.token | string | `""` | github authentiation token value |
| grafana.dashboards.enabled | bool | `true` | Enable the deployment of grafana dashboards |
| grafana.dashboards.label | string | `"grafana_dashboard"` | Label to find dashboards using the k8s sidecar |
| grafana.dashboards.value | string | `"1"` | Label value to find dashboards using the k8s sidecar |
| grafana.folder.annotation | string | `"grafana_folder"` | Annotation to enable folder storage using the k8s sidecar |
| grafana.folder.name | string | `"Policy Reporter"` | Grafana folder in which to store the dashboards |
| grafana.namespace | string | `nil` | namespace for configMap of grafana dashboards |
| image.pullPolicy | string | `"Always"` | The docker image pull policy |
| image.repository | string | `"devopstales/trivy-operator"` | The docker image repository to use |
| image.tag | string | `"2.5.0"` | The docker image tag to use |
| imagePullSecrets | list | `[]` | list of secrets to use for imae pull |
| kube_bench_scnner.image.pullPolicy | string | `"Always"` | The docker image pull policy |
| kube_bench_scnner.image.repository | string | `"devopstales/kube-bench-scnner"` | The docker image repository to use |
| kube_bench_scnner.image.tag | string | `"2.5"` | The docker image tag to use |
| log_level | string | `"INFO"` | Log level |
| monitoring.port | string | `"9115"` | configure prometheus monitoring port |
| namespaceScanner.clusterWide | bool | `false` |  |
| namespaceScanner.crontab | string | `"*/5 * * * *"` |  |
| namespaceScanner.integrations.policyreport | bool | `false` |  |
| namespaceScanner.namespaceSelector | string | `"trivy-scan"` |  |
| nodeSelector | object | `{}` | Set the node selector for the pod. |
| offline.db_repository | string | `"localhost:5000/trivy-db"` | repository to use for download trivy vuln db |
| offline.db_repository_insecure | bool | `false` | insecure repository |
| offline.enabled | bool | `false` | enable air-gapped mode |
| persistence.accessMode | string | `"ReadWriteOnce"` | Volumes mode |
| persistence.annotations | object | `{}` | Volumes annotations |
| persistence.enabled | bool | `true` | Volumes for the pod |
| persistence.size | string | `"1Gi"` | Volumes size |
| podSecurityContext | object | `{"fsGroup":10001,"fsGroupChangePolicy":"OnRootMismatch"}` | security options for the pod |
| registryAuth.enabled | bool | `false` | enable registry authentication |
| registryAuth.image_pull_secrets | list | `["regcred"]` | list of image pull secrets for authentication |
| serviceAccount.annotations | object | `{}` | serviceAccount annotations |
| serviceAccount.create | bool | `true` | Enable serviceAccount creation |
| serviceAccount.name | string | `"trivy-operator"` | Name of the serviceAccount |
| serviceMonitor.enabled | bool | `false` | allow to override the namespace for serviceMonitor |
| serviceMonitor.labels.release | string | `"prometheus"` | labels to match the serviceMonitorSelector of the Prometheus Resource |
| serviceMonitor.metricRelabelings | list | `[]` | metricRelabeling config for serviceMonitor |
| serviceMonitor.namespace | object | `{}` | Name of the namespace for serviceMonitor |
| serviceMonitor.relabelings | list | `[]` | relabel config for serviceMonitor |
| tolerations | list | `[]` | Set the tolerations for the pod. |

```bash
kubectl create ns trivy-operator
kubens trivy-operator
helm upgrade --install trivy devopstales/trivy-operator -f values.yaml
```
