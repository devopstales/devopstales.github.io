---
title: K8S Logging And Monitoring
url: https://devopstales.github.io/kubernetes/k8s-prometheus-stack/
date: 2021-06-15
keywords: kubernetes deployment, kubernetes vs docker, Kubernetes Secuity, best Practices, promethsus, noSQL, loki, grafana
---


In this tutorial I will show you how to install a prometheus operator to monotor kubernetes and loki to gether logs.

<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

Prometheus is an open-source monitoring system with a built-in noSQL time-series database. It offers a multi-dimensional data model, a flexible query language, and diverse visualization possibilities. Prometheus collects metrics from http nedpoint. Most service dind't have this endpoint so you need optional programs that generate additional metrics cald exporters.

### Monitoring

```yaml
nano values.yaml
---
global:
  rbac:
    create: true
    pspEnabled: true

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 10Gi
  ingress:
    enabled: true
    annotations:
      cert-manager.io/cluster-issuer: ca-issuer
    hosts:
      - alertmanager.k8s.intra
    paths:
    - /
    pathType: ImplementationSpecific
    tls:
    - secretName: tls-alertmanager-cert
      hosts:
      - alertmanager.k8s.intra

grafana:
  rbac:
    enable: true
    pspEnabled: true
    pspUseAppArmor: false
  initChownData:
    enabled: false
  enabled: true
  adminPassword: Password1
  plugins:
  - grafana-piechart-panel
  persistence:
    enabled: true
    size: 10Gi
  ingress:
    enabled: true
    annotations:
      cert-manager.io/cluster-issuer: ca-issuer
    hosts:
      - grafana.k8s.intra
    paths:
    - /
    pathType: ImplementationSpecific
    tls:
    - secretName: tls-grafana-cert
      hosts:
      - grafana.k8s.intra


prometheus:
  enabled: true
  prometheusSpec:
    podMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelectorNilUsesHelmValues: false
    secrets: ['etcd-client-cert']
    storageSpec:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 10Gi
  ingress:
    enabled: true
    annotations:
      cert-manager.io/cluster-issuer: ca-issuer
    hosts:
      - prometheus.k8s.intra
    paths:
    - /
    pathType: ImplementationSpecific
    tls:
    - secretName: tls-prometheus-cert
      hosts:
      - prometheus.k8s.intra
```

There is a bug in the Grafana helm chart so it didn't sreate the psp correcly for the init container: https://github.com/grafana/helm-charts/issues/427

```bash
# solution
kubectl edit psp prometheus-grafana
...
  runAsUser:
    rule: RunAsAny
...

kubectl get rs
NAME                                             DESIRED   CURRENT   READY   AGE
prometheus-grafana-74b5d957bc                    1         0         0       12m
...

kubectl delete rs prometheus-grafana-74b5d957bc
```

```bash
#### grafana dashboards
## RKE2
# 14243
## NGINX Ingress controller
# 9614
## cert-manager
# 11001
## longhorn
# 13032
### kyverno
# https://raw.githubusercontent.com/kyverno/grafana-dashboard/master/grafana/dashboard.json
### calico
# 12175
# 3244
### cilium
# 6658
# 14500
# 14502
# 14501
```

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml
```

For the proxy down status:
```bash
kubectl edit cm/kube-proxy -n kube-system
...
kind: KubeProxyConfiguration
metricsBindAddress: 0.0.0.0:10249
...
```

If you use rke2 you can configure this from the helm chart before first start:
```bash
cat << EOF > /var/lib/rancher/rke2/server/manifests/rke2-kube-proxy-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-kube-proxy
  namespace: kube-system
spec:
  valuesContent: |-
    metricsBindAddress: 0.0.0.0:10249
EOF
```

For the controller-manager down status:
```bash
nano /etc/kubernetes/manifests/kube-controller-manager.yaml
# OR
nano /var/lib/rancher/rke2/agent/pod-manifests/kube-controller-manager.yaml
apiVersion: v1
kind: Pod
metadata:
  ...
spec:
  containers:
  - command:
    - kube-controller-manager
    ...
    - --address=0.0.0.0
    ...
    - --bind-address=<your control-plane IP or 0.0.0.0>
    ...
    livenessProbe:
      failureThreshold: 8
      httpGet:
       	host: 0.0.0.0
    ...
```

For the kube-scheduler down status:
```bash
nano /etc/kubernetes/manifests/kube-scheduler.yaml
# OR
nano /var/lib/rancher/rke2/agent/pod-manifests/kube-scheduler.yaml
apiVersion: v1
kind: Pod
metadata:
  ...
spec:
  containers:
  - command:
    - kube-scheduler
    - --address=0.0.0.0
    - --bind-address=0.0.0.0
    ...
    livenessProbe:
      failureThreshold: 8
      httpGet:
       	host: 0.0.0.0
    ...
```

For the etcd down status firs we need to create a secret to authenticate for the etcd:

```bash
# kubeadm
kubectl -n monitoring create secret generic etcd-client-cert \
--from-file=/etc/kubernetes/pki/etcd/ca.crt \
--from-file=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
--from-file=/etc/kubernetes/pki/etcd/healthcheck-client.key

# rancher
kubectl -n monitoring create secret generic etcd-client-cert \
--from-file=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
--from-file=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
--from-file=/var/lib/rancher/rke2/server/tls/etcd/server-client.key
```

Then we configure the prometheus to use it:
```yaml
nano values.yaml
---
...
prometheus:
  enabled: true
  prometheusSpec:
    secrets: ['etcd-client-cert']
...
kubeEtcd:
  enabled: true
  service:
    port: 2379
    targetPort: 2379
    selector:
      component: etcd
  serviceMonitor:
    interval: ""
    scheme: https
    insecureSkipVerify: true
    serverName: ""
    metricRelabelings: []
    relabelings: []
    caFile: /etc/prometheus/secrets/etcd-client-cert/server-ca.crt
    certFile: /etc/prometheus/secrets/etcd-client-cert/server-client.crt
    keyFile: /etc/prometheus/secrets/etcd-client-cert/server-client.key

# for kubeadm
#    caFile: /etc/prometheus/secrets/etcd-client-cert/ca.crt
#    certFile: /etc/prometheus/secrets/etcd-client-cert/healthcheck-client.crt
#    keyFile: /etc/prometheus/secrets/etcd-client-cert/healthcheck-client.key

```

### Monitoring Nginx

```yaml
cat << EOF > /var/lib/rancher/rke2/server/manifests/rke2-ingress-nginx-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-ingress-nginx
  namespace: kube-system
spec:
  valuesContent: |-
    controller:
      metrics:
	      enabled: true
        service:
          annotations:
            prometheus.io/scrape: "true"
            prometheus.io/port: "10254"
        serviceMonitor:
          enabled: true
          namespace: "monitoring"
EOF

kaf /var/lib/rancher/rke2/server/manifests/rke2-ingress-nginx-config.yaml
```

### Monitoring Core-DNS

```yaml
cat << EOF > default-network-dns-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-network-dns-policy
  namespace: kube-system
spec:
  ingress:
  - ports:
    - port: 53
      protocol: TCP
    - port: 53
      protocol: UDP
    - port: 9153
      protocol: TCP
  podSelector:
    matchLabels:
      k8s-app: kube-dns
  policyTypes:
  - Ingress
EOF

kaf default-network-dns-policy.yaml
```

### Monitor cert-manager
```yaml
nano 01-cert-managger.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-system
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: cert-manager
  namespace: ingress-system
spec:
  repo: "https://charts.jetstack.io"
  chart: cert-manager
  targetNamespace: ingress-system
  valuesContent: |-
    installCRDs: true
    clusterResourceNamespace: "ingress-system"
    prometheus:
      enabled: true
      servicemonitor:
        enabled: true
        namespace: "monitoring"

kubectl apply -f 01-cert-managger.yaml
```

### Longhorn

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: longhorn-prometheus-servicemonitor
  namespace: monitoring
  labels:
    name: longhorn-prometheus-servicemonitor
spec:
  selector:
    matchLabels:
      app: longhorn-manager
  namespaceSelector:
    matchNames:
    - longhorn-system
  endpoints:
  - port: manager
```

### Logging

```bash
helm repo add grafana https://grafana.github.io/helm-charts
 
helm repo update
 
helm search repo loki
 
helm upgrade --install loki-stack grafana/loki-stack \
--create-namespace \
--namespace loki-stack \
--set promtail.enabled=true,loki.persistence.enabled=true,loki.persistence.size=10Gi
```

Promtail dose not working with enabled `selinux`, because this promtail deployment store som files on the host filesystem and `selinux` dose not allow to write it, so you need ti use fluent-bit.

```bash
helm upgrade --install loki-stack grafana/loki-stack \
--create-namespace \
--namespace loki-stack \
--set fluent-bit.enabled=true,promtail.enabled=false \
--set loki.persistence.enabled=true,loki.persistence.size=10Gi

```


Add datasource to grafana:
```
type: loki
name: Loki
url: http://loki-stack.loki-stack:3100
```

