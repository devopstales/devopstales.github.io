---
title: Continuous Image Security
url: https://devopstales.github.io/kubernetes/continuous-image-security/
date: 2021-06-15
keywords: kubernetes deployment, kubernetes vs docker, Kubernetes Secuity, best Practices, rancher, Admission Controller, trivy
---


In this post I will show you my tool to Continuously scann deployed images in your Kubernetes cluster.

<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

In a previous posts we talked about admission-controllers that scnas the image at deploy. Like [Banzaicloud's anchore-image-validator](https://devopstales.github.io/home/image-security-admission-controller/) and [Anchore's own admission-controller](https://devopstales.github.io/home/image-security-admission-controller-v2/). But what if you run your image for a long time. Last weak I realised I run containers wit imagest older the a year. I this time period many new vulnerability came up.

I find a tool called [trivy-scanner](https://github.com/fleeto/trivy-scanner) that do almast what I want. It scans the docker images in all namespaces with the label `trivy=true` and get the resoults to a prometheus endpoint. It based on [Shell Operator](https://github.com/flant/shell-operator) that runs a small python script. I made my own version from it:

### Deploy the app

```bash
git clone https://github.com/devopstales/trivy-scanner

nano trivy-scanner/deploy/kubernetes/kustomization.yaml
namespace: trivy-scanner
...

kubectl create ns trivy-scanner
kubectl aplly -k trivy-scanner/deploy/kubernetes/
```

### Demo

Test the `guestbook-demo` namespace:

```bash
kubectl label namespaces guestbook-demo trivy=true

kubectl get service -n trivy-scanner
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)  AGE
trivy-scanner   ClusterIP   10.43.179.39   <none>        9115/TCP   15m

curl -s http://10.43.179.39:9115/metrics | grep so_vulnerabilities
```

Now you need to add the `trivy-scanner` `Service` as target for your prometheus. I created a `ServiceMonitor` object for that:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    serviceapp: trivy-exporter-servicemonitor
    release: prometheus
  name: trivy-exporter-servicemonitor
spec:
  selector:
    matchLabels:
      app: trivy-scanner
  endpoints:
  - port: metrics
```

If you use my grafana dasgboard from the repo you can see someting like this:

![image](/img/include/trivy-exporter.webp) <br>

