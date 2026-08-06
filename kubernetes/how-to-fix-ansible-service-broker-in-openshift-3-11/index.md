---
title: How to fix Ansible Service Broker in OpenShift 3.11
url: https://devopstales.github.io/kubernetes/how-to-fix-ansible-service-broker-in-openshift-3-11/
date: 2020-07-10
keywords: ansible, openshift 3.11, openshift okd, openshift tutorial, openshift container platform, rad hat openshift
---


Ansible Service Broker in OpenShift 3.11 is broken as it uses wrong docker tag.
<!--more-->

{{< content "/filedir/openshift.html" >}}

### Export running deployment to yaml
```
oc project openshift-ansible-service-broker
oc get --export dc/asb -o yaml > asb.yaml
```

### Patch and deploy the yaml

```
replate docker.io/ansibleplaybookbundle/origin-ansible-service-broker:latest to
docker.io/ansibleplaybookbundle/origin-ansible-service-broker:ansible-service-broker-1.3.23-1
oc apply -f asb.yaml


curl -k -H "Authorization: Bearer `oc serviceaccounts get-token asb-client`" https://`oc get routes -n openshift-ansible-service-broker --no-headers | awk '{print $2}'`/osb/v2/catalog
```

### Config for new install
```
ansible_service_broker_image=docker.io/ansibleplaybookbundle/origin-ansible-service-broker:ansible-service-broker-1.3.23-1
```

