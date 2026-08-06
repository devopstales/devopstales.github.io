---
title: OKD OpenShift 4 Monitoring
url: https://devopstales.github.io/kubernetes/openshift4-monitoring/
date: 2023-06-14
keywords: openshift 4.2, openshift okd4, openshift container platform, rad hat openshift, Prometheus, monitoring
---



In this Post I will show you how you can use the enbeddid Prometheus monitoring system in OpenShift 4 to monitor your workload applications.
<!--more-->

{{< content "/filedir/openshift4.html" >}}

### Create Demo app to monitor

First I installed a wordpress by helmfile to use as an app to monitor.

```bash
oc new-project monitoring-test

oc project monitoring-test
```

Deploy test app

```yaml
nano wordpress-helmfile.yaml
---
helmDefaults:
  createNamespace: false

repositories:
- name: bitnami
  url: https://charts.bitnami.com/bitnami

releases:
- name: wordpress-test
  namespace: monitoring-test
  chart: bitnami/wordpress
  set:
  - name: mariadb.primary.containerSecurityContext.enabled
    value: false
  - name: mariadb.primary.podSecurityContext.enabled
    value: false
  - name: mariadb.auth.rootPassword
    value: wordpress-test
  - name: mariadb.primary.persistence.enabled
    value: true
  - name: mariadb.primary.persistence.size
    value: 20Gi
  - name: wordpressPassword
    value: wordpress-test
  - name: persistence.enabled
    value: false
  - name: containerSecurityContext.enabled
    value: false
  - name: podSecurityContext.enabled
    value: false
```

```bash
helmfile apply -f wordpress-helmfile.yaml
```

### Enable user namespace monitoring

By default the enbeddid prometheus only monitor the cluster components. To monitor applications in user namespace you need to enable the user namespace monitoring.

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
    alertmanagerMain:
      enableUserAlertmanagerConfig: true
EOF
```

This will create a separate prometheus stack for user namespace monitoring. List user namespace monitor status:

```bash
oc get pods -n openshift-user-workload-monitoring
NAME                                  READY   STATUS    RESTARTS   AGE
prometheus-operator-7c55995fb-zsjxk   2/2     Running   0          4m
prometheus-user-workload-0            6/6     Running   0          4m
prometheus-user-workload-1            6/6     Running   0          4m
thanos-ruler-user-workload-0          3/3     Running   0          4m
thanos-ruler-user-workload-1          3/3     Running   0          4m
```

Now we can create alert routing to send emails about the alerts.

```bash
oc apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1beta1
kind: AlertmanagerConfig
metadata:
metadata:
  name: pod-num-alert
  namespace: monitoring-test
spec:
  receivers:
    - name: default
    - name: email_alert
      emailConfigs:
      - to: devopstales@
        from: prometheus@mydomain.intra
        smarthost: mail.mydomain.intra:25
        requireTLS: false
        sendResolved: true
  route:
    receiver: email_alert
    groupInterval: 5m
    groupWait: 30s
    repeatInterval: 10m
    groupBy:
      - namespace
    routes:
      - match:
          severity: wordpress
        receiver: email_alert
EOF
```

Create test alert:

```bash
oc apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-num-alert
  namespace: monitoring-test
spec:
  groups:
  - name: monitoring-test
    rules:
    - alert: RunningPodNumAlert
      expr: sum(kube_pod_status_ready{namespace="monitoring-test"}) != 3
      for: 5m
      labels:
        namespace: monitoring-test
        severity: wordpress
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-memory-alert
  namespace: monitoring-test
spec:
  groups:
  - name: monitoring-test
    rules:
    - alert: RunningPodMemoryAlert
      expr: ((( sum(container_memory_working_set_bytes{image!="",container!="POD", namespace="monitoring-test"}) by (namespace,container,pod)))) / 1000000 > 90
      for: 5m
      labels:
        namespace: monitoring-test
        severity: wordpress
EOF
```

