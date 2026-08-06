---
title: How to fix cluster-monitoring-operator in OpenShift 3.11
url: https://devopstales.github.io/kubernetes/how-to-fix-cluster-monitoring-operator-in-openshift-3-11/
date: 2020-07-10
keywords: ansible, openshift 3.11, openshift okd, openshift tutorial, openshift container platform, rad hat openshift
---


Default install use an old image for cluster-monitoring-operator with imagestream false latanci alert  problem.
<!--more-->

{{< content "/filedir/openshift.html" >}}

### Export running deployment to yaml
```
oc project openshift-monitoring
oc get --export deployment/cluster-monitoring-operator -o yaml > cluster-monitoring-operator.yaml
```

### Patch and deploy the yaml

```
sed -i -e "s|image:.*|image: quay.io/openshift/origin-cluster-monitoring-operator:v3.11
|" \
> cluster-monitoring-operator.yaml

oc apply -f cluster-monitoring-operator.yaml


curl -k -H "Authorization: Bearer `oc serviceaccounts get-token asb-client`" https://`oc get routes -n openshift-ansible-service-broker --no-headers | awk '{print $2}'`/osb/v2/catalog

oc delete deployment/prometheus-operator
oc delete statefulset/alertmanager-main
oc delete statefulset/prometheus-k8s
```

### Config for new install
```
openshift_cluster_monitoring_operator_image=quay.io/openshift/origin-cluster-monitoring-operator:v3.11
```

