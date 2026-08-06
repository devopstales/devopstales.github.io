---
title: Openshift Letsencrypt certificates
url: https://devopstales.github.io/kubernetes/openshift-letsencrypt/
date: 2019-05-28
keywords: ansible, openshift 3.11, openshift okd, openshift tutorial, openshift container platform, rad hat openshift, openshift 3.11 letsencrypt, openshift letsencrypt router
---


Thanks to Tomáš Nožička developed openshift-acme as an ACME Controller for OpenShift and Kubernetes clusters. <br>
It automatically provision certficates
<!--more-->

{{< content "/filedir/openshift.html" >}}

### Environment
```
192.168.1.40    deployer
192.168.1.41    openshift01 # master node
192.168.1.42    openshift02 # infra node
192.168.1.43    openshift03 # worker node
```

### Deploy route
```
oc project default

oc label node openshift02.devopstales.intra "router=letsencrypt"
oc get node --show-labels

oc adm policy add-scc-to-user hostnetwork -z router
oc adm router router-letsencrypt --replicas=0 --ports="8080:8080,8443:8443" --stats-port=1937 --selector="router=letsencrypt" --labels="router=letsencrypt"

oc set env dc/router-letsencrypt \
NAMESPACE_LABELS="router=letsencrypt" \
ROUTER_ALLOW_WILDCARD_ROUTES=true \
ROUTER_SERVICE_HTTP_PORT=8080 \
ROUTER_SERVICE_HTTPS_PORT=8443 \
ROUTER_TCP_BALANCE_SCHEME=roundrobin

oc set env dc/router NAMESPACE_LABELS="router != letsencrypt"

oc scale dc/router-letsencrypt --replicas=3
```

### Deploy letsencrypt
```
GIT_REPO=https://raw.githubusercontent.com/devopstales/openshift-examples
GIT_PATH=/master/letsencrypt
oc new-project letsencrypt
oc create -f$GIT_REPO/$GIT_PATH/{clusterrole,serviceaccount,imagestream,deployment}.yaml
oc adm policy add-cluster-role-to-user openshift-acme -z openshift-acme
```

# Demo
```
oc new-project test
oc label namespace test router=letsencrypt
oc new-app centos/ruby-25-centos7~https://github.com/sclorg/ruby-ex.git
oc expose svc/ruby-ex

oc patch route ruby-ex \
    -p '{"metadata":{"annotations":{  "kubernetes.io/tls-acme" : "true"   }}}'
```

