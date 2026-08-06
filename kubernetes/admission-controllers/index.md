---
title: Using Admission Controllers
url: https://devopstales.github.io/kubernetes/admission-controllers/
date: 2020-12-07
keywords: kubernetes deployment, kubernetes vs docker, Kubernetes Secuity, best Practices, Admission Controller
---


In this post I will show you how you can use Admission Controllers.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

### What is an Admission Controller
An admission controller is a piece of code that intercepts requests to the Kubernetes API server prior to persistence of the object, but after the request is authenticated and authorized. […] Admission controllers may be “validating”, “mutating”, or both. Mutating controllers may modify the objects they admit; validating controllers may not. […] If any of the controllers in either phase reject the request, the entire request is rejected immediately and an error is returned to the end-user. (Source: [Kubernetes Website](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) )

In a nutshell, Kubernetes admission controllers are plugins that govern and enforce how the cluster is used. They can be thought of as a gatekeeper that intercepts (authenticated) API requests and may change the request object or deny the request altogether.

### How do I turn on an admission controller?
A list of previously implemented controllers comes with Kubernetes, or you can write your own. To do so you must enable them in the `kube-apiserver` The Kubernetes API server flag enable-admission-plugins takes a comma-delimited list of admission control plugins to invoke prior to modifying objects in the cluster. For example, the following command line enables the NamespaceLifecycle and the LimitRanger admission control plugins:
```
kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger ...
```

### What is an admission webhook?
There are two special admission controllers in the list included in the Kubernetes apiserver: MutatingAdmissionWebhook and ValidatingAdmissionWebhook. These are special admission controllers that send admission requests to external HTTP callbacks and receive admission responses. If these two admission controllers are enabled, a Kubernetes administrator can create and configure an admission webhook in the cluster.

![Example image](/img/include/admission-controller-phases.webp)

Validating webhooks can reject a request, but they cannot modify the object they are receiving in the admission request, while mutating webhooks can modify objects by creating a patch that will be sent back in the admission response. If a webhook rejects a request, an error is returned to the end-user.

### Why do I need admission controllers?
Security: Admission controllers can increase security by mandating a reasonable security baseline across an entire namespace or cluster. The built-in PodSecurityPolicy admission controller is perhaps the most prominent example; it can be used for disallowing containers from running as root or making sure the container’s root filesystem is always mounted read-only, for example. Further use cases that can be realized by custom, webhook-based admission controllers include:

* Allow pulling images only from specific registries known to the enterprise, while denying unknown image registries. 
* Reject deployments that do not meet security standards. For example, containers using the privileged flag can circumvent a lot of security checks. This risk could be mitigated by a webhook-based admission controller that either rejects such deployments (validating) or overrides the privileged flag, setting it to false.


