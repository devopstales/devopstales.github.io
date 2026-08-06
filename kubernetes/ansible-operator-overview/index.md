---
title: Ansible Operator Overview
url: https://devopstales.github.io/kubernetes/ansible-operator-overview/
date: 2019-07-10
keywords: ansible, openshift 3.11, openshift container platform, rad hat openshift, rad hat, configuration management
---


Operators make it easy to manage complex stateful applications on top of Kubernetes. In this pos I will show you how to read an ansible based Openshift operator.

<!--more-->

{{< content "/filedir/openshift.html" >}}

### Why an Operator?

Operators make it easy to manage complex stateful applications on top of Kubernetes. However writing an operator today can be difficult because of challenges such as using low level APIs, writing boilerplate, and a lack of modularity which leads to duplication.

### What is an Ansible Operator?

Ansible Operator is one of the available types of Operators that Operator SDK is able to generate. Operator SDK can create an operator using with Golang, Helm, or Ansible.
It is a collection of building blocks from Operator SDK that enables Ansible to handle the reconciliation logic for an Operator.

### How Ansible Operator works?

We want to trigger this Ansible logic when a Custom Resource changes. The Ansible Operator uses a Watches file, written in YAML, which holds the mapping between Custom Resources and Ansible Roles/Playbooks. The Operator expects this mapping file in a predefined location: `/opt/ansible/watches.yaml`

Each mapping within the Watches file has mandatory fields:

* group: Group of the Custom Resource that you will be watching.
* version: Version of the Custom Resource that you will be watching.
* kind: Kind of the Custom Resource that you will be watching.
* role (default): Path to the Role that should be run by the Operator for a particular Group-Version-Kind (GVK). This field is mutually exclusive with the "playbook" field.
* playbook (optional): Path to the Playbook that should be run by the Operator.

### Initialize new operator template

```
# istall sdk client
brew install operator-sdk

# generate base temlate
operator-sdk new memcached-operator --type=ansible --api-version=cache.example.com/v1alpha1 --kind=Memcached --skip-git-init
cd memcached-operator
```

The sdk cli generated the base structure for all the componets. The main parts are the `watches.yaml` the Dockerfile in `build/Dockerfile` and the ansible role in `roles/`<br> folder. Fore this tutorial I will use this role https://github.com/dymurray/memcached-operator-role.<br>
In the role we must use k8s ansible module to deploy kubernetes compnets to the cluster.<br>

```
nano roles/memcached/tasks/main.yml
---
- name: start memcached
  k8s:
    definition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: '{{ meta.name }}-memcached'
        namespace: '{{ meta.namespace }}'
      spec:
        replicas: "{{size}}"
        selector:
          matchLabels:
            app: memcached
        template:
          metadata:
            labels:
              app: memcached
          spec:
            containers:
            - name: memcached
              command:
              - memcached
              - -m=64
              - -o
              - modern
              - -v
              image: "memcached:1.4.36-alpine"
              ports:
                - containerPort: 11211
```
You can use variables in ansible Jinja temlate from the CR spec or from kubernetes enviroment like namespace: `{{ meta.namespace }}`.<br>
Add default value to the `{{size}}` variable:

```
nano roles/memcached/defaults/main.yml
---
size: 1
```

#### Variable sharing Example
```
apiVersion: "foo.example.com/v1alpha1"
kind: "Foo"
metadata:
  name: "example"
annotations:
  ansible.operator-sdk/reconcile-period: "30s"
  name: "example"
spec:
  message: "Hello world 2"
  newParameter: "newParam"
# associates GVK with Role
role: /opt/ansible/roles/Foo
```

```
- debug:
    msg: "message value from CR spec: {{ message }}"

- debug:
    msg: "newParameter value from CR spec: {{ new_parameter }}"

- debug:
    msg: "name: {{ meta.name }}, namespace: {{ meta.namespace }}"
```


The Openshidt SDK created a simple Dockerfile in `build/Dockerfile` to run the newly created ansible role. We nead to build a docker image from this Dockerfile and use this image in our memcached-operator deployment.

```
# buid image
operator-sdk build memcached-operator:v0.0.1

# Edit deploy/operator.yaml to use the newly created memcached-operator:v0.0.1 docker image
sed -i 's|{{ REPLACE_IMAGE }}|memcached-operator:v0.0.1|g' deploy/operator.yaml

# If we did not want to download the image (besause we build it on the worker or it is representid on all of my workes) we can disable image pulling.
sed -i "s|{{ pull_policy\|default('Always') }}|Never|g" deploy/operator.yaml
```

### Creating the Operator from deploy manifests
```
oc create -f deploy/crds/cache_v1alpha1_memcached_crd.yaml

oc new-project tutorial
oc create -f deploy/service_account.yaml
oc create -f deploy/role.yaml
oc create -f deploy/role_binding.yaml
oc create -f deploy/operator.yaml

oc get deployment
```

Now that we have deployed our Operator, let's create a CR and deploy an instance of memcached.
There is a sample CR in the scaffolding created as part of the Operator SDK. Inspect `deploy/crds/cache_v1alpha1_memcached_cr.yaml`, and then use it to create a Memcached custom resource.

```
oc create -f deploy/crds/cache_v1alpha1_memcached_cr.yaml



$ oc get deployment
NAME                 DESIRED CURRENT UP-TO-DATE AVAILABLE AGE
memcached-operator   1       1       1          1         2m
example-memcached    3       3       3          3         1m

#Check the pods to confirm 3 replicas were created:

$ oc get pods
NAME                                READY STATUS   RESTARTS AGE
example-memcached-6cc844747c-2hbln  1/1   Running  0        1m
example-memcached-6cc844747c-54q26  1/1   Running  0        1m
example-memcached-6cc844747c-7jfhc  1/1   Running  0        1m
memcached-operator-68b5b558c5-dxjwh 1/1   Running  0        2m
```

### Removing Memcached from the cluster

```
# First, delete the 'memcached' CR, which will remove the 4 Memcached pods and the associated deployment.
oc delete -f deploy/crds/cache_v1alpha1_memcached_cr.yaml

# Then, delete the memcached-operator deployment.
oc delete -f deploy/operator.yaml
```

