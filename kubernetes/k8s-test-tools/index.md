---
title: Validate Kubernetes Deployment in CI/CD
url: https://devopstales.github.io/kubernetes/k8s-test-tools/
date: 2022-03-02
keywords: kubernetes deployment, kubernetes vs docker, Kubernetes Secuity, Best Practices, CI, CD, Gitlab, Trivy
---


I this blog post I will show you how you can validate your kubernetes objects, helm charts, images at CI/CD.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

### The yaml
First this is the example yaml that we will use for validation tests.

```yaml
nano base-valid.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-echo
spec:
  selector:
    matchLabels:
      app: http-echo
  template:
    metadata:
      labels:
        app: http-echo
    spec:
      containers:
      - name: http-echo
        image: jxlwqq/http-echo:latest
        args: ["-text", "hello-world"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: http-echo
spec:
  ports:
  - port: 5678
    protocol: TCP
    targetPort: 5678
  selector:
    app: http-echo
```

### kubeval

`kubeval` is a tool for validating a Kubernetes YAML or JSON configuration file. It does so using schemas generated from the Kubernetes OpenAPI specification, and therefore can validate schemas for multiple versions of Kubernetes.

```bash
brew install kubeval
```

```bash
kubeval base-valid.yaml
PASS - base-valid.yaml contains a valid Deployment (http-echo)
PASS - base-valid.yaml contains a valid Service (http-echo)
```

```bash
kubeval --kubernetes-version 1.16.1 base-valid.yaml
```

> One limitation of kubeval is that it is currently not able to validate against Custom Resource Definitions (CRDs)

### kube-score

`Kube-score` analyses YAML manifests and scores them against security recommendations and best practices.

```bash
brew install kube-score
``

```bash
kube-score score base-valid.yaml
apps/v1/Deployment http-echo
    [CRITICAL] Container Resources
        · http-echo -> CPU limit is not set
            Resource limits are recommended to avoid resource DDOS. Set resources.limits.cpu
        · http-echo -> Memory limit is not set
            Resource limits are recommended to avoid resource DDOS. Set resources.limits.memory
        · http-echo -> CPU request is not set
            Resource requests are recommended to make sure that the application can start and run without crashing. Set resources.requests.cpu
        · http-echo -> Memory request is not set
            Resource requests are recommended to make sure that the application can start and run without crashing. Set
            resources.requests.memory
    [CRITICAL] Pod NetworkPolicy
        · The pod does not have a matching NetworkPolicy
            Create a NetworkPolicy that targets this pod to control who/what can communicate with this pod. Note, this feature needs to be
            supported by the CNI implementation used in the Kubernetes cluster to have an effect.
    [CRITICAL] Pod Probes
        · Container is missing a readinessProbe
            A readinessProbe should be used to indicate when the service is ready to receive traffic. Without it, the Pod is risking to
            receive traffic before it has booted. It's also used during rollouts, and can prevent downtime if a new version of the application
            is failing.
            More information: https://github.com/zegl/kube-score/blob/master/README_PROBES.md
    [CRITICAL] Container Security Context User Group ID
        · http-echo -> Container has no configured security context
            Set securityContext to run the container in a more secure context.
    [CRITICAL] Container Image Tag
        · http-echo -> Image with latest tag
            Using a fixed tag is recommended to avoid accidental upgrades
    [CRITICAL] Container Ephemeral Storage Request and Limit
        · http-echo -> Ephemeral Storage limit is not set
            Resource limits are recommended to avoid resource DDOS. Set resources.limits.ephemeral-storage
    [CRITICAL] Container Security Context ReadOnlyRootFilesystem
        · http-echo -> Container has no configured security context
            Set securityContext to run the container in a more secure context.
    [WARNING] Deployment has host PodAntiAffinity
        · Deployment does not have a host podAntiAffinity set
            It's recommended to set a podAntiAffinity that stops multiple pods from a deployment from being scheduled on the same node. This
            increases availability in case the node becomes unavailable.
    [CRITICAL] Deployment has PodDisruptionBudget
        · No matching PodDisruptionBudget was found
            It's recommended to define a PodDisruptionBudget to avoid unexpected downtime during Kubernetes maintenance operations, such as
            when draining a node.
v1/Service http-echo
```

If you plan to use it as part of your Continuous Integration pipeline, you can use a more concise output with the flag `--output-format ci`.

```bash
kube-score score base-valid.yaml --output-format ci
[OK] http-echo apps/v1/Deployment
[OK] http-echo apps/v1/Deployment
[CRITICAL] http-echo apps/v1/Deployment: (http-echo) CPU limit is not set
[CRITICAL] http-echo apps/v1/Deployment: (http-echo) Memory limit is not set
[CRITICAL] http-echo apps/v1/Deployment: (http-echo) CPU request is not set
[CRITICAL] http-echo apps/v1/Deployment: (http-echo) Memory request is not set
[CRITICAL] http-echo apps/v1/Deployment: (http-echo) Image with latest tag
[CRITICAL] http-echo apps/v1/Deployment: (http-echo) Ephemeral Storage limit is not set
[CRITICAL] http-echo apps/v1/Deployment: Container is missing a readinessProbe
[OK] http-echo apps/v1/Deployment
[CRITICAL] http-echo apps/v1/Deployment: The pod does not have a matching NetworkPolicy
[CRITICAL] http-echo apps/v1/Deployment: (http-echo) Container has no configured security context
[OK] http-echo apps/v1/Deployment
[CRITICAL] http-echo apps/v1/Deployment: (http-echo) Container has no configured security context
[CRITICAL] http-echo apps/v1/Deployment: No matching PodDisruptionBudget was found
[WARNING] http-echo apps/v1/Deployment: Deployment does not have a host podAntiAffinity set
[SKIPPED] http-echo apps/v1/Deployment: Skipped because the deployment is not targeted by a HorizontalPodAutoscaler
[OK] http-echo apps/v1/Deployment
[OK] http-echo v1/Service
[OK] http-echo v1/Service
[OK] http-echo v1/Service
[OK] http-echo v1/Service
```

### trivy

`Trivy` (`tri` pronounced like trigger, `vy` pronounced like envy) is a simple and comprehensive scanner for vulnerabilities in container images. 

```bash
trivy image jxlwqq/http-echo:latest

jxlwqq/http-echo:latest (debian 11.1)
=====================================
Total: 0 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0)


http-echo (gobinary)
====================
Total: 0 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0)
```

It also provides built-in policies to detect configuration issues in Docker, Kubernetes and Terraform. Also, you can write your own policies in `Rego` to scan JSON, YAML, HCL, etc, like Conftest.

```bash
trivy config base-valid.yaml
2022-03-09T18:38:49.725+0100	INFO	Detected config files: 1

base-valid.yaml (kubernetes)
============================
Tests: 39 (SUCCESSES: 28, FAILURES: 11, EXCEPTIONS: 0)
Failures: 11 (UNKNOWN: 0, LOW: 7, MEDIUM: 4, HIGH: 0, CRITICAL: 0)

+---------------------------+------------+----------------------------------------+----------+--------------------------------------------+
|           TYPE            | MISCONF ID |                 CHECK                  | SEVERITY |                  MESSAGE                   |
+---------------------------+------------+----------------------------------------+----------+--------------------------------------------+
| Kubernetes Security Check |   KSV001   | Process can elevate its own privileges |  MEDIUM  | Container 'http-echo' of                   |
|                           |            |                                        |          | Deployment 'http-echo' should set          |
|                           |            |                                        |          | 'securityContext.allowPrivilegeEscalation' |
|                           |            |                                        |          | to false                                   |
|                           |            |                                        |          | -->avd.aquasec.com/appshield/ksv001        |
+                           +------------+----------------------------------------+----------+--------------------------------------------+
|                           |   KSV003   | Default capabilities not dropped       |   LOW    | Container 'http-echo' of Deployment        |
|                           |            |                                        |          | 'http-echo' should add 'ALL' to            |
|                           |            |                                        |          | 'securityContext.capabilities.drop'        |
|                           |            |                                        |          | -->avd.aquasec.com/appshield/ksv003        |
+                           +------------+----------------------------------------+          +--------------------------------------------+
|                           |   KSV011   | CPU not limited                        |          | Container 'http-echo' of                   |
|                           |            |                                        |          | Deployment 'http-echo' should              |
|                           |            |                                        |          | set 'resources.limits.cpu'                 |
|                           |            |                                        |          | -->avd.aquasec.com/appshield/ksv011        |
+                           +------------+----------------------------------------+----------+--------------------------------------------+
|                           |   KSV012   | Runs as root user                      |  MEDIUM  | Container 'http-echo' of                   |
|                           |            |                                        |          | Deployment 'http-echo' should set          |
|                           |            |                                        |          | 'securityContext.runAsNonRoot' to true     |
|                           |            |                                        |          | -->avd.aquasec.com/appshield/ksv012        |
+                           +------------+----------------------------------------+----------+--------------------------------------------+
|                           |   KSV013   | Image tag ':latest' used               |   LOW    | Container 'http-echo' of Deployment        |
|                           |            |                                        |          | 'http-echo' should specify an image tag    |
|                           |            |                                        |          | -->avd.aquasec.com/appshield/ksv013        |
+                           +------------+----------------------------------------+          +--------------------------------------------+
|                           |   KSV014   | Root file system is not read-only      |          | Container 'http-echo' of                   |
|                           |            |                                        |          | Deployment 'http-echo' should set          |
|                           |            |                                        |          | 'securityContext.readOnlyRootFilesystem'   |
|                           |            |                                        |          | to true                                    |
|                           |            |                                        |          | -->avd.aquasec.com/appshield/ksv014        |
+                           +------------+----------------------------------------+          +--------------------------------------------+
|                           |   KSV015   | CPU requests not specified             |          | Container 'http-echo' of                   |
|                           |            |                                        |          | Deployment 'http-echo' should              |
|                           |            |                                        |          | set 'resources.requests.cpu'               |
|                           |            |                                        |          | -->avd.aquasec.com/appshield/ksv015        |
+                           +------------+----------------------------------------+          +--------------------------------------------+
|                           |   KSV016   | Memory requests not specified          |          | Container 'http-echo' of                   |
|                           |            |                                        |          | Deployment 'http-echo' should              |
|                           |            |                                        |          | set 'resources.requests.memory'            |
|                           |            |                                        |          | -->avd.aquasec.com/appshield/ksv016        |
+                           +------------+----------------------------------------+          +--------------------------------------------+
|                           |   KSV018   | Memory not limited                     |          | Container 'http-echo' of                   |
|                           |            |                                        |          | Deployment 'http-echo' should              |
|                           |            |                                        |          | set 'resources.limits.memory'              |
|                           |            |                                        |          | -->avd.aquasec.com/appshield/ksv018        |
+                           +------------+----------------------------------------+----------+--------------------------------------------+
|                           |   KSV020   | Runs with low user ID                  |  MEDIUM  | Container 'http-echo' of                   |
|                           |            |                                        |          | Deployment 'http-echo' should set          |
|                           |            |                                        |          | 'securityContext.runAsUser' > 10000        |
|                           |            |                                        |          | -->avd.aquasec.com/appshield/ksv020        |
+                           +------------+----------------------------------------+          +--------------------------------------------+
|                           |   KSV021   | Runs with low group ID                 |          | Container 'http-echo' of                   |
|                           |            |                                        |          | Deployment 'http-echo' should set          |
|                           |            |                                        |          | 'securityContext.runAsGroup' > 10000       |
|                           |            |                                        |          | -->avd.aquasec.com/appshield/ksv021        |
+---------------------------+------------+----------------------------------------+----------+--------------------------------------------+
```

You can integrate trivy to your ci/cd pipeline by using ine of the output template:

```bash
export TRIVY_VERSION=0.24.2
wget --no-verbose https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz -O - | tar -zxvf -

trivy image --exit-code 0 --no-progress --format template --template "@contrib/gitlab.tpl" -o gl-container-scanning-report.json golang:1.12-alpine

trivy image --exit-code 0 --no-progress --format template --template "@contrib/junit.tpl" -o junit-report.xml  golang:1.12-alpine
```

