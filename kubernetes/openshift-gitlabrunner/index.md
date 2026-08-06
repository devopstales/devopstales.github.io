---
title: Install Gitlab runner on Openshift
url: https://devopstales.github.io/kubernetes/openshift-gitlabrunner/
date: 2019-04-20
keywords: ansible, openshift 3.11, openshift okd, openshift tutorial, openshift container platform, rad hat openshift, gitlab org chart, gitlab cicd, gitlab runners, gitlab runner docker
---


In this post I will configure a gtlab rubber for Openshift.

<!--more-->

### Creating a Service Account
```
oc new-project gitlab-rubber
oc create sa gitlab-ci
oc policy add-role-to-user edit system:serviceaccount:gitlab-rubber:gitlab-ci

oc get sa
NAME         SECRETS   AGE
builder      2         2d
default      2         2d
deployer     2         2d
gitlab-ci    2         2d

oc describe sa gitlab-ci
Name:           gitlab-ci
Namespace:      constellation
Labels:         <none>
Annotations:    <none>

Image pull secrets:     gitlab-ci-dockercfg-q5mj9

Mountable secrets:      gitlab-ci-token-gvvkv
                        gitlab-ci-dockercfg-q5mj9

Tokens:                 gitlab-ci-token-gvvkv
                        gitlab-ci-token-tfsf7

oc describe secret gitlab-ci-token-gvvkv
...
token:          eyJ...<very-long-token>...-cw

oc login --token=eyJ...<very-long-token>...-cw
```

### Edit Gitlab-ci config
```
nano  .gitlab-ci.yml
image: ebits/openshift-client

stages:
  - deployToOpenShift

variables:
  OPENSHIFT_SERVER: https://master.openshift.devopstales.intra:443
  OPENSHIFT_DOMAIN: openshift.devopstales.intra
  # Configure this variable in Secure Variables:
  OPENSHIFT_TOKEN: eyJ...<very-long-token>...-cw

.deploy: &deploy
  before_script:
    - oc login "$OPENSHIFT_SERVER" --token="$OPENSHIFT_TOKEN" --insecure-skip-tls-verify
  # login with the service account
    - oc project "slides-openshift"
  # enter into our slides project on OpenShift
  script:
    - "oc get services $APP 2> /dev/null || oc new-app . --name=$APP"
  # create a new application from the image in the OpenShift registry
    - "oc start-build $APP --from-dir=. --follow || sleep 3s"
  # start a new build
    - "oc get routes $APP 2> /dev/null || oc expose service $APP --hostname=$APP_HOST"
  # expose our application

develop:
  <<: *deploy
  stage: deployToOpenShift
  tags:
    - docker
  variables:
    APP: slides-openshift
    APP_HOST: demo-slides.$OPENSHIFT_DOMAIN
  environment:
    name: develop
    url: http://demo-slides.$OPENSHIFT_DOMAIN
  except:
    - master
```

### Create a kubernetes runner in Openshift from template:
```
wget https://raw.githubusercontent.com/devopstales/openshift-examples/master/template/gitlab-runner-template.yml
oc deploy gitlab-runner-template.yml
```

Deploy the template from the gui:

```
oc adm policy add-scc-to-user privileged system:serviceaccount:gitlab-rubber:<application-name>-user
```

