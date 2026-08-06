---
title: trivy-operator 2.4: Patch release for Admisssion controller
url: https://devopstales.github.io/kubernetes/trivy-operator-2.4/
date: 2022-06-30
keywords: Kubernetes, trivy-operator, trivy, operator, Image scan, image security
---


Today I am happy to announce the release of trivy-operator 2.4. This blog post focuses on the functionality provided by the trivy-operator 2.4 release. 

<!--more-->

### What is trivy-operator?

Trivy-operator is a Kubernetes Operator based on the open-source container vulnerability scanner Trivy. The goal of this project is to provide a vulnerability scanner that continuously scans containers deployed in a Kubernetes cluster. [Built with Kubernetes Operator Pythonic Framework (Kopf)](https://github.com/nolar/kopf) There are a few solution for checking the images when you deploy them to the Kubernetes cluster, but fighting against vulnerabilities is a day to day task. Check once is not enough when every day is a new das for frats. That is why I created trivy-operator so you can create scheduled image scans on your running pods.

### What is new

With the release of trivy-operator 2.4 ther is the fallowin new features:

* add policyreport creation and Policy Reporter UI integration
* update vulnerabilityreports if exists
* add ownerReferences for VulnerabilityReport
* add redis cache
* add [documentation website](https://devopstales.github.io/trivy-operator/)

### PolicyReport

The [PolicyReport](https://github.com/kubernetes-sigs/wg-policy-prototypes/tree/master/policy-report) object is a prototype object probosed by the Kubernetes policy work group. The Policy Report Custom Resource Definition (CRD) can be used as a common way to provide policy results to Kubernetes cluster administrators and users, using native tools. See the [proposal](https://docs.google.com/document/d/1nICYLkYS1RE3gJzuHOfHeAC25QIkFZfgymFjgOzMDVw/edit#) for background and details.

Add the PolicyReport CRDs to your cluster (v1alpha2): 

```bash
kubectl create -f https://github.com/kubernetes-sigs/wg-policy-prototypes/raw/master/policy-report/crd/v1alpha2/wgpolicyk8s.io_policyreports.yaml
```

If you installed the trivy-operator by the helm chart the Policy Report Custom Resource Definition is installed automticle.

This objects can be visualized by the Policy Reporter UI. 

### Policy Reporter UI integration
The Policy Reporter UI is a monitoring and Observability Tool for the PolicyReport CRD with an optional UI. It is created by Kyverno. The main goal was a tool to visualize the resoluts of the Kyverno policies, but because it uses the PolicyReports CRD it can visualize the resoults of the trivy-operator scans.

Install the Policy Reporter UI:

```bash
helm repo add policy-reporter https://kyverno.github.io/policy-reporter
helm repo update

helm install policy-reporter policy-reporter/policy-reporter \
--set kyvernoPlugin.enabled=true --set ui.enabled=true --set ui.plugins.kyverno=true \
-n policy-reporter --create-namespace

kubectl port-forward service/policy-reporter-ui 8082:8080 -n policy-reporter
```

Open `http://localhost:8082/` in your browser.

![Policy Reporter UI](/img/include/policy_report.webp)

### Trivy Image Validator

The admission controller function can be configured as a ValidatingWebhook in a k8s cluster. Kubernetes will send requests to the admission server when a Pod creation is initiated. The admission controller checks the image using trivy if it is in a namespace with the label `trivy-operator-validation=true`.

You can define policy to the Admission Controller, by adding annotation to the pod trough the deployment:

```yaml
spec:
  ...
  template:
    metadata:
      annotations:
        trivy.security.devopstales.io/medium: "5"
        trivy.security.devopstales.io/low: "10"
        trivy.security.devopstales.io/critical: "2"
...
```

### Where you can find:
With the release of trivy-operator 2.4 I published trivy-operator with OperatorFramework to OperatorHub:

![OperatorHub](/img/include/trivy-operator-OH.webp)

![OperatorHub](/img/include/trivy-operator-OH2.webp)

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

|               Parameter             |                Description                  |                  Default                 |
| ----------------------------------- | ------------------------------------------- | -----------------------------------------|
| image.repository                    | image | devopstales/trivy-operator |
| image.pullPolicy                    | pullPolicy | Always |
| image.tag                           | image tag | 2.1 |
| imagePullSecrets                    | imagePullSecrets list | [] |
| podSecurityContext.fsGroup          | mount id | 10001 |
| serviceAccount.create               | create serviceAccount | true |
| serviceAccount.annotations          | add annotation to serviceAccount | {} |
| serviceAccount.name                 | name of the serviceAccount | trivy-operator |
| monitoring.port                     | prometheus endpoint port | 9115 |
| serviceMonitor.enabled              | enable serviceMonitor object creation | false |
| serviceMonitor.namespace            | where to create serviceMonitor object | kube-system |
| storage.enabled                     | enable pv to store trivy database | true |
| storage.size                        | pv size | 1Gi |
| NamespaceScanner.crontab            | cronjob scheduler | "*/5 * * * *" |
| NamespaceScanner.namespaceSelector  | Namespace Selector | "trivy-scan" |
| registryAuth.enabled                | enable registry authentication in operator | false |
| registryAuth.registry               | registry name for authentication |
| registryAuth.user                   | username for authentication |
| registryAuth.password               | password for authentication |
| githubToken.enabled                 | Enable githubToken usage for trivy database update | false |
| githubToken.token                   | githubToken value | "" |

```bash
kubectl create ns trivy-operator
kubens trivy-operator
helm upgrade --install trivy devopstales/trivy-operator -f values.yaml
```

### Monitoring

Trivy-operatos has a prometheus endpoint op port `9115` and can be deployed wit `ServiceMonitor` for automated scrapping.

```bash
curl -s http://10.43.179.39:9115/metrics | grep trivy_vulnerabilities
# HELP trivy_vulnerabilities_sum Container vulnerabilities
# TYPE trivy_vulnerabilities_sum gauge
trivy_vulnerabilities_sum{exported_namespace="trivytest",image="docker.io/openshift/mysql-56-centos7:latest",severity="scanning_error"} 1.0
trivy_vulnerabilities_sum{exported_namespace="trivytest",image="docker.io/nginxinc/nginx-unprivileged:latest",severity="UNKNOWN"} 0.0
trivy_vulnerabilities_sum{exported_namespace="trivytest",image="docker.io/nginxinc/nginx-unprivileged:latest",severity="LOW"} 83.0
trivy_vulnerabilities_sum{exported_namespace="trivytest",image="docker.io/nginxinc/nginx-unprivileged:latest",severity="MEDIUM"} 5.0
trivy_vulnerabilities_sum{exported_namespace="trivytest",image="docker.io/nginxinc/nginx-unprivileged:latest",severity="HIGH"} 7.0
trivy_vulnerabilities_sum{exported_namespace="trivytest",image="docker.io/nginxinc/nginx-unprivileged:latest",severity="CRITICAL"} 4.0
trivy_vulnerabilities_sum{exported_namespace="trivytest",image="docker.io/library/nginx:1.18",severity="UNKNOWN"} 0.0
trivy_vulnerabilities_sum{exported_namespace="trivytest",image="docker.io/library/nginx:1.18",severity="LOW"} 126.0
trivy_vulnerabilities_sum{exported_namespace="trivytest",image="docker.io/library/nginx:1.18",severity="MEDIUM"} 25.0
trivy_vulnerabilities_sum{exported_namespace="trivytest",image="docker.io/library/nginx:1.18",severity="HIGH"} 43.0
trivy_vulnerabilities_sum{exported_namespace="trivytest",image="docker.io/library/nginx:1.18",severity="CRITICAL"} 21.0
# HELP trivy_vulnerabilities Container vulnerabilities
# TYPE trivy_vulnerabilities gauge
trivy_vulnerabilities{exported_namespace="trivytest",image="docker.io/nginxinc/nginx-unprivileged:latest",installedVersion="2.3.4",pkgName="pkgName",severity="LOW",vulnerabilityId="CVE-2011-3374"} 1.0
trivy_vulnerabilities{exported_namespace="trivytest",image="docker.io/nginxinc/nginx-unprivileged:latest",installedVersion="8.32-4",pkgName="pkgName",severity="LOW",vulnerabilityId="CVE-2016-2781"} 1.0
trivy_vulnerabilities{exported_namespace="trivytest",image="docker.io/nginxinc/nginx-unprivileged:latest",installedVersion="8.32-4",pkgName="pkgName",severity="LOW",vulnerabilityId="CVE-2017-18018"} 1.0
trivy_vulnerabilities{exported_namespace="trivytest",image="docker.io/nginxinc/nginx-unprivileged:latest",installedVersion="7.74.0-1.3",pkgName="pkgName",severity="CRITICAL",vulnerabilityId="CVE-2021-22945"} 1.0
trivy_vulnerabilities{exported_namespace="trivytest",image="docker.io/nginxinc/nginx-unprivileged:latest",installedVersion="7.74.0-1.3",pkgName="pkgName",severity="HIGH",vulnerabilityId="CVE-2021-22946"} 1.0
trivy_vulnerabilities{exported_namespace="trivytest",image="docker.io/nginxinc/nginx-unprivileged:latest",installedVersion="7.74.0-1.3",pkgName="pkgName",severity="MEDIUM",vulnerabilityId="CVE-2021-22947"} 1.0
trivy_vulnerabilities{exported_namespace="trivytest",image="docker.io/nginxinc/nginx-unprivileged:latest",installedVersion="7.74.0-1.3",pkgName="pkgName",severity="LOW",vulnerabilityId="CVE-2021-22898"} 1.0
```

![Grafana Dashboard](/img/include/trivy-exporter.webp)
