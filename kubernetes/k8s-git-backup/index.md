---
title: How to Backup Kubernetes to git?
url: https://devopstales.github.io/kubernetes/k8s-git-backup/
date: 2021-08-28
keywords: Kubernetes, k8s, Backup, git
---


In this tutorial I will show you how you can backup the kubernetes object to git as yaml-s.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

Thanky to [Maxim Levchenko](https://github.com/WoozyMasta) ther is a grate tool called [kube-dump](https://github.com/WoozyMasta/kube-dump) that is dump all of the kubernetes objects to a git repository as yaml. We will use this tool to backup.

Key features:

* Saving is done only for those resources to which you have read access.
* You can pass a list of namespaces as an input, otherwise all available for your context will be used.
* Both namespace resources and global cluster resources are subject to persistence.
* You can use the utility locally as a regular script or run it in a container or in a kubernetes cluster, for example, as a CronJob.
* It can create archives and rotate them after itself.
* Can commit state to git repository and push to remote repository.
* You can specify a specific list of cluster resources for unloading.

```bash
kubectl create ns kube-dump
kubectl -n kube-dump apply -f \
  https://raw.githubusercontent.com/WoozyMasta/kube-dump/master/deploy/cluster-role-view.yaml
```

### Deploy with git repository oauth token

> Project access tokens are supported for self-managed instances on Free and above. They are also supported on GitLab SaaS Premium and above. If you use GitLab SaaS on Free you can us Personal access token instead of Project Access Token.

As an example, I will use authorization in GitLab using the Project Access Token, so we will create a secret with the repository address and an authorization token:

```bash
kubectl -n kube-dump create secret generic kube-dump \
  --from-literal=GIT_REMOTE_URL=https://oauth2:$TOKEN@corp-gitlab.com/devops/cluster-01.git
```

> Before Kubernetes 1.22 CronJob's timezone is always UTC. If you want to change this use [cronjobber](https://github.com/hiddeco/cronjobber)
> Since Kubernetes 1.22 you can add [timezon in cronjob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) with `CRON_TZ` variable.

Let’s set up a CronJob in which we indicate the frequency of the task launch:

```bash
wget https://github.com/WoozyMasta/kube-dump/blob/master/deploy/cronjob-git-token.yaml

nano cronjob-git-token.yaml
...
spec:
  schedule: "0 1 * * *"

kubectl apply -f cronjob-git-token.yaml -n kube-dump
```

### Deploy with git repository write allowed ssh key

Generate ssh key:

```bash
mkdir -p ./.ssh
chmod 0700 ./.ssh
ssh-keygen -t ed25519 -C "kube-dump" -f ./.ssh/kube-dump
cat ./.ssh/kube-dump.pub

kubectl -n kube-dump create secret generic kube-dump-key \
  --from-file=./.ssh/kube-dump \
  --from-file=./.ssh/kube-dump.pub
```

Create pvc for store data such as cache:

```bash
kubectl apply -n kube-dump -f deploy/pvc.yaml
```

And apply the cron job manifest, previously you could set up environment variables:

```bash
wget https://github.com/WoozyMasta/kube-dump/blob/master/deploy/cronjob-git-key.yaml

nano cronjob-git-key.yaml
...
spec:
  schedule: "0 1 * * *"
...
              env:
                - name: MODE
                  value: "dump"
                - name: DESTINATION_DIR
                  value: "/data/dump"
                - name: GIT_PUSH
                  value: "true"
                - name: GIT_BRANCH
                  value: "master"
                - name: GIT_COMMIT_USER
                  value: "Kube Dump"
                - name: GIT_COMMIT_EMAIL
                  value: "kube@dump.local"
                - name: GIT_REMOTE_URL
                  value: "git@corp-gitlab.com:devops/cluster-bkp.git"

kubectl apply -f cronjob-git-key.yaml -n kube-dump
```
