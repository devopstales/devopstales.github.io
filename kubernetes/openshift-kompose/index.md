---
title: Openshift: Run docker-compoe in Openshift
url: https://devopstales.github.io/kubernetes/openshift-kompose/
date: 2019-09-08
keywords: ansible, openshift 3.11, openshift okd, openshift tutorial, openshift container platform, rad hat openshift, kompose
---



Kompose is a open source tool that uses docker-compose file to deploy on kubernetes. Openshift is also Kubernetes based and Kompose is support Openshift too.
<!--more-->

{{< content "/filedir/openshift.html" >}}

Let’s say we have project with multiple microsevices that needs to deploy on Openshift and they have docker-compose.yml and Dockerfile. How do we deploy on Openshift?<br>

Use Kompose to convert the docker-compose file to openshift compatible kubernetes config.

```
kompose convert -f docker-compose.yaml --provider=openshift
```

This command creates a separet yaml file for all kubernetes building block like  "-imagestream.yaml", "-service.yaml", "-deploymentconfig.yaml" for each microservice. We can use these config files to deploy on Openshift easily.

```
kompose up --provider=openshift -f docker-compose.yml --build build-config --namespace=devopstales
```

If you have "build" option in docker-compose file, kompose automatically detects remote git url to deploy automatically.

