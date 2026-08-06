---
title: Openshift secondary route
url: https://devopstales.github.io/kubernetes/openshift-secondary-router/
date: 2019-12-20
keywords: ansible, openshift 3.11, openshift okd, openshift tutorial, openshift container platform, rad hat openshift
---


I this tutorial I will show you how to create a secondari router for Openshift.
<!--more-->

{{< content "/filedir/openshift.html" >}}

### Environment
```
192.168.1.40    deployer
192.168.1.41    openshift01 # master node
192.168.1.42    openshift02 # infra node
192.168.1.43    openshift03 # worker node with second router
192.168.1.44    openshift04 # worker node
```

### Deploy route
```
oc adm router router-public --replicas=2 --ports="8080:8080,8443:8443" \
--stats-port=1937 --selector="router=public" --labels="router=public"

oc set env dc/router-public \
DEFAULT_CERTIFICATE_PATH=/etc/pki/tls/private/tls.crt \
NAMESPACE_LABELS="router=public" \
ROUTER_ALLOW_WILDCARD_ROUTES=true \
ROUTER_ENABLE_HTTP2=true \
ROUTER_HAPROXY_CONFIG_MANAGER=true \
ROUTER_SERVICE_HTTP_PORT=8080 \
ROUTER_SERVICE_HTTPS_PORT=8443 \
ROUTER_TCP_BALANCE_SCHEME=roundrobin

oc label node openshift03 "router=public"
```

Configurate your firewall to create a NAT rule from publicIP:80 to openshift03:8080 and publicIP:443 to openshift03:8443

### Demo
```
oc new-project test
oc label namespace test router=public
```

