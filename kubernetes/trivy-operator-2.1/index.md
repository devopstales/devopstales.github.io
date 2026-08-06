---
title: trivy-operator 2.1: Trivy-operator is now an Admisssion controller too!!!
url: https://devopstales.github.io/kubernetes/trivy-operator-2.1/
date: 2021-12-17
keywords: Kubernetes, trivy-operator, trivy, operator, Image scan, image security
---


Today I am happy to announce the release of trivy-operator 2.1. This blog post focuses on the functionality provided by the trivy-operator 2.1 release. 

<!--more-->

### What is trivy-operator?

Trivy-operator is a Kubernetes Operator based on the open-source container vulnerability scanner Trivy. The goal of this project is to provide a vulnerability scanner that continuously scans containers deployed in a Kubernetes cluster. [Built with Kubernetes Operator Pythonic Framework (Kopf)](https://github.com/nolar/kopf) There are a few solution for checking the images when you deploy them to the Kubernetes cluster, but fighting against vulnerabilities is a day to day task. Check once is not enough when every day is a new das for frats. That is why I created trivy-operator so you can create scheduled image scans on your running pods.

### New Functions

With the release of trivy-operator 2.1 It is not just an operator but an Admisssion controller too!!! Now you can check the Images at deploy time with the same tool. You didn't need to deploy multiple applications for image scanning.

### Trivy Image Validator

The admission controller function can be configured as a `ValidatingWebhook` in a k8s cluster. Kubernetes will send requests to the admission server when a Pod creation is initiated. The admission controller checks the image using trivy.

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
With the release of trivy-operator 2.1 I published trivy-operator with OperatorFramework to OperatorHub:

![OperatorHub](/img/include/trivy-operator-OH.webp)


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
  tag: "2.1"

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
curl -s http://10.43.179.39:9115/metrics | grep so_vulnerabilities

so_vulnerabilities{exported_namespace="trivytest",image="docker.io/library/nginx:1.18",severity="UNKNOWN"} 0
so_vulnerabilities{exported_namespace="trivytest",image="docker.io/library/nginx:1.18",severity="LOW"} 23
so_vulnerabilities{exported_namespace="trivytest",image="docker.io/library/nginx:1.18",severity="MEDIUM"} 93
so_vulnerabilities{exported_namespace="trivytest",image="docker.io/library/nginx:1.18",severity="HIGH"} 76
so_vulnerabilities{exported_namespace="trivytest",image="docker.io/library/nginx:1.18",severity="CRITICAL"} 25
so_vulnerabilities{exported_namespace="trivytest",image="docker.io/library/nginx:latest",severity="UNKNOWN"} 0
so_vulnerabilities{exported_namespace="trivytest",image="docker.io/library/nginx:latest",severity="LOW"} 23
so_vulnerabilities{exported_namespace="trivytest",image="docker.io/library/nginx:latest",severity="MEDIUM"} 88
so_vulnerabilities{exported_namespace="trivytest",image="docker.io/library/nginx:latest",severity="HIGH"} 60
so_vulnerabilities{exported_namespace="trivytest",image="docker.io/library/nginx:latest",severity="CRITICAL"} 8

```
