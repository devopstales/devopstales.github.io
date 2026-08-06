---
title: Configure OKD OpenShift 4 ingress
url: https://devopstales.github.io/kubernetes/openshift4-ingress/
date: 2021-12-13
keywords: openshift 4.2, openshift okd4, openshift container platform, rad hat openshift
---



In this Post I will show you how you can create multiple ingress route on an OpenShift 4 on premise.
<!--more-->

{{< content "/filedir/openshift4.html" >}}

### What is Ingress Operator

Ingress Operator is an OpenShift component which enables external access to cluster services by configuring Ingress Controllers, which route traffic as specified by OpenShift Route and Kubernetes Ingress resources.

To provide this functionality, Ingress Operator deploys and manages an OpenShift router — a HAProxy-based Kubernetes ingress controller.

### Add default certificate for Ingress Operator

Create the secret containing the certificate:

```bash
cat tls.crt | base64
LS0tLS1CRUdJTiBDRVJUSUZ...
cat tls.key | base64
LS0tLS1CRUdJTiBQUklWQVR...
```

> Always use the full certificate chain for thi ingress secret

```yaml
nano cert.yaml
---
kind: Secret
apiVersion: v1
metadata:
  name: default-tls-cert
  namespace: openshift-ingress
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZ...
  tls.key: LS0tLS1CRUdJTiBQUklWQVR...
type: Opaque
---
kind: Secret
apiVersion: v1
metadata:
  name: default-tls-cert
  namespace: openshift-config
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZ...
  tls.key: LS0tLS1CRUdJTiBQUklWQVR...
type: Opaque
```

### Create multiple ingress route

For the example I will create a private and a public rout for the cluster

```yaml
nano default.yaml
---
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  defaultCertificate:
    name: default-tls-cert
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/ingress-internal: ''
  replicas: 2
```

```yaml
nano public.yaml
---
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: public
  namespace: openshift-ingress-operator
spec:
  defaultCertificate:
    name: default-tls-cert
  domain: external.okd.mydomain.intra
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/ingress-public: ''
  replicas: 2
  routeSelector:
    matchLabels:
      router: public
```

```bash
oc label nodes okd4-worker-1 node-role.kubernetes.io/ingress-internal=""
oc label nodes okd4-worker-2 node-role.kubernetes.io/ingress-internal=""
oc label nodes okd4-worker-3 node-role.kubernetes.io/ingress-public=""
oc label nodes okd4-worker-4 node-role.kubernetes.io/ingress-public=""

oc apply -f cert.yaml
oc apply -f ingress/
oc patch ingresscontroller.operator default --type=merge -p '{"spec":{"defaultCertificate": {"name": "default-tls-cert"}}}' -n openshift-ingress-operator
oc patch consoles.operator.openshift.io cluster --type=merge -p '{"spec":{"route":{"secret":{"name": "default-tls-cert"}}}}' -n openshift-config
```

### enable HTTP/2

```bash
oc annotate ingresses.config/cluster ingress.operator.openshift.io/default-enable-http2=true
```

### Add OKD 3.11 type conole url

I used OKD 3.11 and ther the conosle usrl wa master.okd.mydomain.intra so I desided to create the same route for okd4:

```yaml
nano master-okf-mydomain-intra.yaml
---
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: master-okd-mydomain-intra
  namespace: openshift-console
  labels:
    app: console
spec:
  host: master.okd.mydomain.intra
  to:
    kind: Service
    name: console
    weight: 100
  port:
    targetPort: https
  tls:
    termination: reencrypt
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: None
```

