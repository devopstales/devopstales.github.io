---
title: Restrict access to OpenShift routes by IP address
url: https://devopstales.github.io/kubernetes/openshift-restrict-access/
date: 2019-09-20
keywords: ansible, openshift 3.11, openshift container platform, rad hat openshift, vmware
---


In this post I will show you how can restrict access to the routes by source IP address.

<!--more-->

{{< content "/filedir/openshift.html" >}}

### Restricting access to a route
After creating and exposing a route, you can add an annotation to the route specifying the IP address(es) that you would like to whitelist. Whitelisting a IP address automatically blacklists everything else.

```
oc annotate route test-route haproxy.router.openshift.io/ip_whitelist=192.168.0.0/24
```

To allow several IP addresses through to the route, separate each IP with a space:

```
oc annotate route test-route haproxy.router.openshift.io/ip_whitelist=192.168.1.10 180.5.61.153 192.168.1.0/24 192.168.0.0/24
```

To delete the IPs from the annotation, you can run the command:

```
oc annotate route test-route haproxy.router.openshift.io/ip_whitelist-
```

