---
title: Troubleshooting Kubernetes: API error at resource listing
url: https://devopstales.github.io/kubernetes/debug-couldnt-get-resource-list/
date: 2023-07-10
keywords: Troubleshooting, Kubernetes, couldn't get resource list for, API error at resource listing, kubernetes, deployment
---


I this post I will show you how to troubleshoot your Kubernetes cluster when you get an API error at resource listing.
<!--more-->

Yesterday when I tried to use my Kubernetes cluster, I got an error:

```bash
couldn't get resource list for custom.metrics.k8s.io/v1beta1: the server is currently unable to handle the request
```

I din't understand first so I started debugging. T

The `custom.metrics.k8s.io/v1beta1` is not a standard Kubernetes api. Kubernetes API can be extended with Custom Resources that are fully managed by Kubernetes and available to Kubectl and other tools. When you get it wit `kubectl` the Kubernetes API will find it like any other API object in etcd. The other way is to use a third-party API server living in the Kubernetes cluster as a pod to serv the object. So when you get it wit `kubectl` the Kubernetes API server will forward the request to the third-party API server. For this forward to work you need an `APIService` object:

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: metrics-server
    namespace: kube-system
    port: 443
  version: v1beta1
  versionPriority: 100
```

First I believed it is the CRD of the metrics-server, and I started debugging the metrics-server. Check its logs but I found no problem. The I find the `APIService` object and realized this is not the same object.

```bash
kubectl get apiservice
NAME                                   SERVICE                                                          AVAILABLE                 AGE
v1.                                    Local                                                            True                      3y362d
v1.acme.cert-manager.io                Local                                                            True                      131d
v1.admissionregistration.k8s.io        Local                                                            True                      3y4d
v1.apiextensions.k8s.io                Local                                                            True                      3y4d
v1.apps                                Local                                                            True                      3y362d
v1.authentication.k8s.io               Local                                                            True                      3y362d
v1.authorization.k8s.io                Local                                                            True                      3y362d
v1.autoscaling                         Local                                                            True                      3y362d
v1.autoscaling.k8s.io                  Local                                                            True                      79d
v1.batch                               Local                                                            True                      3y362d
v1.cert-manager.io                     Local                                                            True                      79d
v1.certificates.k8s.io                 Local                                                            True                      623d
v1.coordination.k8s.io                 Local                                                            True                      3y362d
...
v1beta1.custom.metrics.k8s.io          cattle-monitoring-system/rancher-monitoring-prometheus-adapter   False (ServiceNotFound)   679d
...
v1beta1.metrics.k8s.io                 kube-system/metrics-server                                       True                      2y36d
```

The information we're looking for on this table is all the `APIServices` where **Available** is **False**. So I figured out the problem is caused by the removal og rancher, that left behind som object at deletion.

## Conclusion

One of the most common cause of the 'couldn't get resource list for' error is the Kubernetes Metrics object served by metrics-server or prometheus-adapter, because metrics are not stored in Etcd like other resources. They are stored in memory or in prometheus, and then exposed via an API extension. If the API is not available, you will get this error.

