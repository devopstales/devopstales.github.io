---
title: Tekton Basics
url: https://devopstales.github.io/devops/tecton-basics/
date: 2024-11-16
---


In this post I will show you how to Install and configure Tekton on a Kubernetes cluster.

<!--more-->

### What is Tekton

Tekton is an open-source cloud native CICD (Continuous Integration and Continuous Delivery/Deployment) solution.

### Overview

Let's visualize what we want to perform in our CI/CD flow. First, we will make changes to our codebase. Those changes will be pushed to a GitHub repository. The push event on the GitHub webhook will trigger our pipeline.

![Tekton](/img/include/tekton101.webp)

### Installing Tekton Operator 

Install Tekton From Yaml.

```bash
kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml
```

If you get an error abaut kubernetes version:

```json
{"severity":"fatal","timestamp":"2024-11-17T09:09:14.017Z","logger":"controller","caller":"sharedmain/main.go:391","message":"Version check failed","commit":"c6d2a8d","error":"kubernetes version \"1.27.3\" is not compatible, need at least \"1.28.0-0\" (this can be overridden with the env var \"KUBERNETES_MIN_VERSION\")"}
```

Edit the falloging pods:

```bash
kubectl edit deployments.apps -n tekton-pipelines tekton-pipelines-controller
kubectl edit deployments.apps -n tekton-pipelines tekton-pipelines-webhook
kubectl edit deployments.apps -n tekton-pipelines tekton-triggers-controller
kubectl edit deployments.apps -n tekton-pipelines tekton-triggers-webhook
kubectl edit deployments.apps -n tekton-pipelines tekton-events-controller
...
        env:
        - name: KUBERNETES_MIN_VERSION
          value: 1.27.0
...
```

### Handling Github Events 

To get external enevts from github we will create an `EventListener`. This will create a service exposed via Kubernetes API to recove github webhook events.

> Change `{{ .Values.projectName }}` with your github project name.

```yaml
---
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: tekton-github-pr-{{ .Values.projectName }}
spec:
  serviceAccountName: service-account-{{ .Values.projectName }}
  triggers:
    - name: pr-trigger
      interceptors:
        - ref:
            name: "cel"
            kind: ClusterInterceptor
            apiVersion: triggers.tekton.dev
          params:
            - name: "filter"
              value: >
                header.match('x-github-event', 'merge')
            - name: "overlays"
              value:
                - key: author
                  expression: body.pusher.name.lowerAscii().replace('/','-').replace('.', '-').replace('_', '-')
                - key: pr-ref
                  expression: body.ref.lowerAscii().replace("/", '-')
      bindings:
        - ref: tb-github-pr-trigger-binding-{{ .Values.projectName }}
      template:
        ref: tt-github-pr-trigger-template-{{ .Values.projectName }}
```

We will use a Cel ClusterInterceptor, another custom resource so we can write filter expressions using CEL, This is how we manage to evaluate the webhook request and filter triggers for many kinds of pipelines.

### Bindings

TriggerBindings are another way to bind objects from the webhook request to variables we can use to control pipeline flow.

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: tb-github-pr-trigger-binding-{{ .Values.projectName }}
spec:
  params:
    - name: revision
      value: $(body.after)
    - name: repo-url
      value: $(body.repository.ssh_url)
    - name: author
      value: $(extensions.author)
    - name: pr-ref
      value: $(extensions.pr-ref)
    - name: repo-full-name
      value: $(body.repository.full_name)
```

### Triggering the Pipeline

TriggerTemplate is the resource that pieces together events with the variables we set up on the TriggerBinding. Here we will associate variables as params to the pipelines, creating a PipelineRun, which is the actual automation being executed as a pod in Kubernetes.

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: tt-github-pr-trigger-template-{{ .Values.projectName }}
spec:
  params:
    - name: revision
    - name: repo-url
    - name: author
    - name: repo-full-name
    - name: pr-ref
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: pr-$(tt.params.pr-ref)-$(tt.params.author)-
      spec:
        serviceAccountName: service-account-{{ .Values.projectName }}
        pipelineRef:
          name: {{ .Values.projectName}}-pipeline
        workspaces:
          - name: cache
            persistentVolumeClaim:
              claimName: pvc-cache-{{ .Values.projectName }}
          - name: shared-data
            volumeClaimTemplate:
              spec:
                accessModes:
                  - ReadWriteOnce
                resources:
                  requests:
                    storage: 2Gi
        params:
          - name: repo-url
            value: $(tt.params.repo-url)
          - name: revision
            value: $(tt.params.revision)
          - name: repo-full-name
            value: $(tt.params.repo-full-name)
          - name: ref
            value: $(tt.params.ref)
          - name: deploy-staging
            value: $(tt.params.deploy-staging)
          - name: test-all
            value: $(tt.params.test-all)
```

### Create Pipeline

The pipeline is the orchestrated flow of tasks we want to run. It will have the parameters we defined in the TriggerTemplate and the logic we want to execute applied to tasks. For this example, we will look at a simple test pipeline that clones a repository and runs the test script.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: {{ .Values.projectName }}-pipeline-tests
spec:
  workspaces:
    - name: shared-data
    - name: cache
  params:
    - name: repo-url
      type: string
    - name: revision
      type: string
  tasks:
    - name: fetch-source
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: task-git-clone
          - name: namespace
            value: tekton-pipelines
      params:
        - name: url
          value: $(params.repo-url)
        - name: revision
          value: $(params.revision)
        - name: depth
          value: 2
      workspaces:
        - name: output
          workspace: shared-data
    - name: install-deps
      runAfter: ["fetch-source"]
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: task-install-deps
          - name: namespace
            value: tekton-pipelines
      params: 
        - name: install-script
          value: yarn install --prefer-offline --ignore-engines
      workspaces:
        - name: source
          workspace: shared-data
        - name: cache
          workspace: cache
    - name: test-task
      runAfter: ["install-deps"]
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: task-test
          - name: namespace
            value: tekton-pipelines
      params:
        - name: diff
          value: $(tasks.fetch-source.results.diff)
        - name: install-deps
          value: yarn install
        - name: run-test
          value: yarn test
      workspaces:
        - name: source
          workspace: shared-data
        - name: cache
          workspace: cache
```

Here we organize the logic of our pipeline.

To help with organization and reutilization of tasks, which are the more atomic resources of a Tekton pipeline, we use a cluster resolver. This way, we can have one task shared across all namespaces and eliminate the need to duplicate tasks that are common to multiple pipelines. The cluster resolver takes the namespace the task is in and the name of the task.

The parameters we define in the TriggerTemplate and pass to the pipeline run are defined in the pipeline and passed to the tasks.

Another great feature of the Tekton pipeline is the TaskResult. Notice we use a parameter in the test-task that is inherited from a task result. This result is defined in the task fetch-source, which is the task we will create to clone a remote repository. The parameter diff is a list of files that were modified in the PR that triggered this pipeline.

The workspaces we define in the TriggerTemplate are also assigned to the tasks. This ensures all pods created for all tasks in the pipeline execute our automations in the same storage space. That way, we can clone the remote repository at the beginning of the pipeline and perform many tasks with the same files.

### Creating our tasks

Now we define the tasks, which are the actual work to be done in our pipeline. In Tekton, each task is a Pod in Kubernetes. It is composed of several steps, each step being a container inside this task pod.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: task-test
  namespace: {{ .Values.projectName }}
spec:
  description: >-
    A generic task to run any bash command in any given image
  workspaces:
    - name: source
      optional: true
    - name: cache
      optional: true
  params:
    - name: run-test
      type: string
    - name: install-deps
      type: string
    - name: diff
      type: string
      description: diff of the pull request
    - name: image
      type: string
      default: "node:latest"
  steps:
    - name: install
      image: $(params.image)
      workingDir: $(workspaces.source.path)
      script: |
         #!/usr/bin/env bash
         set -xe
         $(params.install-deps)
    - name: test
      image: $(params.image)
      workingDir: $(workspaces.source.path)
      script: |
        #!/usr/bin/env bash
        set -xe
        $(params.run-test)
```

This is a simple task that executes a script command you provide. As with the previous task, we define the workspace where we clone the repository and define one step to install dependencies and another to run the tests. There are many ways to organize this same scenario; this is just an example of tasks and how steps are defined.

### Dashboard

In order to visualize your tekton resources, We will install Tekton Dashboard:

```bash
kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml
```

```bash
kubectl get services -n tekton-pipelines

NAME                                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                              AGE
el-tekton-github-pr-devopstales     ClusterIP   10.1.226.52    <none>        8080/TCP,9000/TCP                    6m56s
tekton-events-controller            ClusterIP   10.1.176.181   <none>        9090/TCP,8008/TCP,8080/TCP           33m
tekton-pipelines-controller         ClusterIP   10.1.11.101    <none>        9090/TCP,8008/TCP,8080/TCP           33m
tekton-pipelines-webhook            ClusterIP   10.1.253.10    <none>        9090/TCP,8008/TCP,443/TCP,8080/TCP   33m
tekton-triggers-controller          ClusterIP   10.1.197.45    <none>        9000/TCP                             33m
tekton-triggers-core-interceptors   ClusterIP   10.1.69.207    <none>        8443/TCP                             33m
tekton-triggers-webhook             ClusterIP   10.1.193.0     <none>        443/TCP                              33m

kubectl --namespace tekton-pipelines port-forward svc/tekton-dashboard 9097:9097
```

### Using an Ingress rule

A more advanced solution is to expose the Dashboard through an Ingress rule.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tekton-dashboard
  namespace: tekton-pipelines
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: domain.tld
    http:
      paths:
      - path: /tekton(/|$)(.*)
        backend:
          service:
            name: tekton-dashboard
            port:
              number: 9097
```
