---
title: CrowdSec Intrusion Detection System (IDS) for Kubernetes
url: https://devopstales.github.io/kubernetes/k8s-crowdsec-ids/
date: 2022-07-08
keywords: Kubernetes, CrowdSec, Intrusion Detection System, IDS
---


In this post I will show you how you can install CrowdSec Intrusion Detection System (IDS) inside a Kubernetes cluster.

<!--more-->

### Prerequisites

* installes Kubernetes cluster
* installed nginx ingress controller
* kubectl
* helm

### Architecture

Here’s an architecture overview of CrowdSec inside a K8s cluster.

![CrowdSec Architecture](/img/include/crowdsec_in_k8s.webp)

### Install CrowdSec

```bash
helm repo add crowdsec https://crowdsecurity.github.io/helm-charts
helm repo update

kubectl create ns crowdsec
kubens crowdsec
```

We can create a new file `crowdsec-values.yaml`, containing the CrowdSec chart configuration.

```yaml
nano crowdsec-values.yaml
---
agent:
  # To specify each pod you want to process it logs (pods present in the node)
  acquisition:
    # The namespace where the pod is located
    - namespace: ingress-nginx
      # The pod name
      podName: ingress-nginx-controller-*
      # as in crowdsec configuration, we need to specify the program name so the parser will match and parse logs
      program: nginx
  # Those are ENV variables
  env:
  # As it's a test, we don't want to share signals with CrowdSec so disable the Online API.
  - name: DISABLE_ONLINE_API
    value: "true"
  # As we are running Nginx, we want to install the Nginx collection
  - name: COLLECTIONS
    value: "crowdsecurity/nginx"
lapi:
  env:
    # As it's a test, we don't want to share signals with CrowdSec, so disable the Online API.
    - name: DISABLE_ONLINE_API
      value: "true"
```

Now we can install CrowdSec using our config file in the CrowdSec namespace we created previously.

```bash
helm install crowdsec crowdsec/crowdsec -f crowdsec-values.yaml -n crowdsec
kubectl get pods -n crowdsec
```

### Demo

Install HelloWorld application:

```bash
helm install helloworld crowdsec/helloworld

echo "192.168.200.100 helloworld.mydomain.intra" | sudo tee -a /etc/hosts

curl -v http://helloworld.mydomain.intra

* Trying 192.168.200.100:80...
* TCP_NODELAY set
* Connected to helloworld.mydomain.intra (192.168.200.100) port 80 (#0)
> GET / HTTP/1.1
> Host: helloworld.mydomain.intra
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Mon, 20 Sep 2021 10:38:21 GMT
< Content-Type: text/plain; charset=utf-8
< Content-Length: 13
< Connection: keep-alive
< X-App-Name: http-echo
< X-App-Version: 0.2.3
<
helloworld!
* Connection #0 to host helloworld.mydomain.intra left intact
```

To test whether CrowdSec detects attacks, we will simulate an attack on the HelloWorld application using Nikto and see CrowdSec metrics and alerts.

```bash
./nikto.pl -host http://helloworld.mydomain.inta
```

Now we can get a shell into the CrowdSec agent pod and see metrics and alerts:

```bash
kubectl get pods -n crowdsec
kubectl -n crowdsec exec -it crowdsec-agent-vn4bp -- sh
```

```bash
/ # cscli metrics
INFO[21-06-2022 09:39:50 AM] Buckets Metrics:                             
+-------------------------------------------+---------------+-----------+--------------+--------+---------+
|                  BUCKET                   | CURRENT COUNT | OVERFLOWS | INSTANCIATED | POURED | EXPIRED |
+-------------------------------------------+---------------+-----------+--------------+--------+---------+
| crowdsecurity/http-bad-user-agent         |             3 |       183 |          186 |    369 | -       |
| crowdsecurity/http-crawl-non_statics      | -             |         7 |            9 |    351 |       2 |
| crowdsecurity/http-path-traversal-probing | -             | -         |            1 |      2 |       1 |
| crowdsecurity/http-probing                |             1 | -         |            2 |      2 |       1 |
| crowdsecurity/http-sensitive-files        | -             |         3 |            4 |     17 |       1 |
+-------------------------------------------+---------------+-----------+--------------+--------+---------+
INFO[21-06-2022 09:39:50 AM] Acquisition Metrics:                         
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+--------------+----------------+------------------------+
|                                                                             SOURCE                                                                              | LINES READ | LINES PARSED | LINES UNPARSED | LINES POURED TO BUCKET |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+--------------+----------------+------------------------+
| file:/var/log/containers/ingress-nginx-controller-fd7bb8d66-llxc9_ingress-nginx_controller-c536915796f13bbf66d1a8ab7159dbd055773dbbf89ab4d9653043591dfaef1f.log |        371 |          371 | -              |                    741 |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+--------------+----------------+------------------------+
INFO[21-06-2022 09:39:50 AM] Parser Metrics:                              
+--------------------------------+------+--------+----------+
|            PARSERS             | HITS | PARSED | UNPARSED |
+--------------------------------+------+--------+----------+
| child-crowdsecurity/http-logs  | 1113 |    738 |      375 |
| child-crowdsecurity/nginx-logs |  371 |    371 | -        |
| crowdsecurity/dateparse-enrich |  371 |    371 | -        |
| crowdsecurity/docker-logs      |  371 |    371 | -        |
| crowdsecurity/geoip-enrich     |  371 |    371 | -        |
| crowdsecurity/http-logs        |  371 |    360 |       11 |
| crowdsecurity/nginx-logs       |  371 |    371 | -        |
| crowdsecurity/whitelists       |  371 |    371 | -        |
+--------------------------------+------+--------+----------+
```

### Install Crowdsec Lua bouncer plugin

Naow we can integrate CrowdSec with nginx ingress controller to block atackers.

```bash
kubectl -n crowdsec exec -it crowdsec-agent-vn4bp -- sh

cscli bouncers add ingress-nginx

Api key for 'ingress-nginx':

   e00b2155a7e43dd8e8d9294305bd9741

Please keep this key since you will not be able to retrieve it!
```

You will get an API key, you need to keep it and save it for the ingress-nginx bouncer. Now we can patch our ingress-nginx helm chart to add and enable the crowdsec lua plugin using the following configuration:

```yaml
nano crowdsec-ingress-bouncer.yaml
---
controller:
  extraVolumes:
  - name: crowdsec-bouncer-plugin
    emptyDir: {}
  extraInitContainers:
  - name: init-clone-crowdsec-bouncer
    image: crowdsec-lua
    imagePullPolicy: IfNotPresent
    env:
      - name: API_URL
        value: "http://crowdsec-service.crowdsec.svc.cluster.local:8080"
      - name: API_KEY
        value: "e00b2155a7e43dd8e8d9294305bd9741"
      - name: DISABLE_RUN
        value: "true"
      - name: BOUNCER_CONFIG
        value: "/crowdsec/crowdsec-bouncer.conf"
    command: ['sh', '-c', "sh /docker_start.sh; mkdir -p /lua_plugins/crowdsec/; cp /crowdsec/* /lua_plugins/crowdsec/"]
    volumeMounts:
    - name: crowdsec-bouncer-plugin
      mountPath: /lua_plugins
  extraVolumeMounts:
  - name: crowdsec-bouncer-plugin
    mountPath: /etc/nginx/lua/plugins/crowdsec
    subPath: crowdsec
  config:
    plugins: "crowdsec"
    lua-shared-dicts: "crowdsec_cache: 50m"
```

Once we have this patch we can upgrade the ingress-nginx chart:

```bash
helm -n ingress-system upgrade \
-f ingress-nginx-values.yaml \
-f crowdsec-ingress-bouncer.yaml \
ingress-nginx ingress-nginx/ingress-nginx
```

### Demo 2

Now we have our ingress controller patched with CrowdSec Lua bouncer plugin. We'll start an attack again using Nikto 

```bash
./nikto.pl -host http://helloworld.mydomain.intra
```

Getting a shell in the CrowdSec agent pod and listing the alerts:

```bash
 # cscli decisions list
+----+----------+-------------------+--------------------------------------+--------+---------+-------------+--------+--------------------+----------+
| ID |  SOURCE  |    SCOPE:VALUE    |                REASON                | ACTION | COUNTRY |     AS      | EVENTS |     EXPIRATION     | ALERT ID |
+----+----------+-------------------+--------------------------------------+--------+---------+-------------+--------+--------------------+----------+
|  3 | crowdsec | Ip:192.168.200.5  | crowdsecurity/http-crawl-non_statics | ban    | FR      | 0123 Orange |     43 | 3h59m44.053908518s |        3 |
+----+----------+-------------------+--------------------------------------+--------+---------+-------------+--------+--------------------+----------+
```

Now, if we try to access the helloworld app using CURL

```bash
$ curl -v http://helloworld.mydomain.intra
*   Trying 192.168.200.100:80...
* TCP_NODELAY set
* Connected to helloworld.mydomain.intra (192.168.200.100) port 80 (#0)
> GET / HTTP/1.1
> Host: helloworld.mydomain.intra
> User-Agent: curl/7.68.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< Date: Mon, 27 Dec 2021 16:14:26 GMT
< Content-Type: text/html
< Content-Length: 146
< Connection: keep-alive
< 

403 Forbidden
nginx

* Connection #0 to host helloworld.mydomain.intra left intact
```

To make the app accessible again, from the crowdsec-agent pod, we just need to delete the decision on our IP.

```bash
$ cscli decisions delete --ip 192.168.200.5
INFO[21-06-2022 04:17:10 PM] 4 decision(s) deleted
```

And CURL the helloworld app again.

```bash
$ curl -v http://helloworld.mydomain.intra
*   Trying 192.168.200.100:80...
* TCP_NODELAY set
* Connected to helloworld.mydomain.intra (192.168.200.100) port 80 (#0)
> GET / HTTP/1.1
> Host: helloworld.mydomain.intra
> User-Agent: curl/7.68.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Mon, 27 Dec 2021 16:18:17 GMT
< Content-Type: text/plain; charset=utf-8
< Content-Length: 13
< Connection: keep-alive
< X-App-Name: http-echo
< X-App-Version: 0.2.3
< 
helloworld !
* Connection #0 to host helloworld.mydomain.intra left intact
```
