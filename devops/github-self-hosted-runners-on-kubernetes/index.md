---
title: Github Self-Hosted Runners on Kubernetes with Actions Runner Controller
url: https://devopstales.github.io/devops/github-self-hosted-runners-on-kubernetes/
date: 2024-01-29
keywords: devops, devops tools, GitHub, ci/cd, Continuous Deployments, Continuous Integration, artifacts management, GitHub Actions, GitHub pipelines
---


In this post I will show you how you can set up self-hosted GitHub action runner in Kubernetes with Actions Runner Controller.

<!--more-->

Following are the steps to set up an organization-level container-based runner within an EKS cluster.

### GitHub authentication for Action Runner Controller

First, we need to set up a mechanism to authenticate the action runner controller to GitHub. This can be done in two ways:

* PAT (Personal Access Token)
* Using GitHub App

In this demo I will use the GithubApp method for authentication.

* In the upper-right corner of any page on GitHub, click your profile photo.
* Navigate to your account settings.
  * For a GitHub App owned by a personal account, click Settings.
  * For a GitHub App owned by an organization:
    * Click Your organizations.
    * To the right of the organization, click Settings.
* In the left sidebar, click Developer settings.
* In the left sidebar, click GitHub Apps.
* Click New GitHub App.

![Example image](/img/include/github-app01.webp)

Set the fallowing permissions:

* Repository Permissions
  * Actions (read only)
  * Administration (read and write)
  * Checks (read only)
  * Metadata (read only)
  * Pull requests (read only)
* Organization Permissions
  * Self-hosted runners (read/write)
  * Webhooks(read and write)

You will get an `App ID` `Client ID` and `Client Secret`. This will be used later.

Download the private key file by pushing the “Generate a private key” button at the bottom of the GitHub App page for later use.

When the installation is complete, you will be taken to a URL. The last number of the URL will be used as the Installation ID later (for example, if the URL ends in settings/installations/12345, the Installation ID is 12345).

Finally, register the App ID (APP_ID), Installation ID (INSTALLATION_ID), and the downloaded private key file (PRIVATE_KEY_FILE_PATH) as Kubernetes secrets using the following command:

```bash
kubectl create secret generic arc-secret -n arc-systems \
 - from-literal=github_app_id=YOUR_GITHUB_APP_ID \
 - from-literal=github_app_installation_id=YOUR_INSTALLATION_ID \
 - from-literal=github_app_private_key=YOUR_PRIVATE_KEY
```

### Install the Action Runner Controller (ARC)

To install the operator and the custom resource definitions (CRDs) in your cluster, do the following.

```bash
helm install arc \
    --namespace arc-system \
    --create-namespace \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

> For additional Helm configuration options, see [values.yaml](https://github.com/actions/actions-runner-controller/blob/master/charts/gha-runner-scale-set-controller/values.yaml) in the ARC documentation.

### Configuring a runner scale set

To configure your runner scale set, run the following command in your terminal, using values from your ARC configuration.

> As a security best practice, install your runner scale sets to a different namespace than the namespace containing your Action Runner Controller.

```bash
helm install arc-runner-set \
    --namespace arc-systems \
    --create-namespace \
    --set githubConfigUrl="https://github.com/<your_enterprise/org/" \
    --set githubConfigSecret.githubConfigSecret="arc-secret" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

> For more Helm configuration options, see [values.yaml](https://github.com/actions/actions-runner-controller/blob/master/charts/gha-runner-scale-set/values.yaml) in the ARC documentation.

From your terminal, run the following command to check your installation.

```bash
helm list -A
```

You should see an output similar to the following.

```bash
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                                       APP VERSION
arc             arc-systems     1               2023-04-12 11:45:59.152090536 +0000 UTC deployed        gha-runner-scale-set-controller-0.4.0       0.4.0
arc-runner-set  arc-runners     1               2023-04-12 11:46:13.451041354 +0000 UTC deployed        gha-runner-scale-set-0.4.0                  0.4.0
```

To check the manager pod, run the following command in your terminal.

```bash
kubectl get pods -n arc-systems
```

If everything was installed successfully, the status of the pods shows as Running.

```bah
NAME                                                   READY   STATUS    RESTARTS   AGE
arc-gha-runner-scale-set-controller-594cdc976f-m7cjs   1/1     Running   0          64s
arc-runner-set-754b578d-listener                       1/1     Running   0          12s
```

### Using runner scale sets

Now you will create and run a simple test workflow that uses the runner scale set runners.

```yaml
name: Actions Runner Controller Demo
on:
  workflow_dispatch:

jobs:
  Explore-GitHub-Actions:
    # You need to use the INSTALLATION_NAME from the previous steps
    runs-on: arc-runner-set
    steps:
    - run: echo "🎉 This job uses runner scale set runners!"
```

Once you've added the workflow to your repository, manually trigger the workflow.  To view the runner pods being created while the workflow is running, run the following command from your terminal.

```bash
kubectl get pods -n arc-runners
```

A successful output will look similar to the following.

```bash
NAMESPACE     NAME                                                  READY   STATUS    RESTARTS      AGE
arc-runners   arc-runner-set-rmrgw-runner-p9p5n                     1/1     Running   0             21s
```
