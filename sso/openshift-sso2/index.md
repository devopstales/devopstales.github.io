---
title: Openshift SSO with Gitlab
url: https://devopstales.github.io/sso/openshift-sso2/
date: 2019-06-17
keywords: Keycloak, oauth2, SSO, OIDC, openshift
---


Configure Openshift Cluster to use Gitlab as a user backend for login with oauth2 and SSO.

<!--more-->

{{< content "/filedir/openshift.html" >}}

With Ansible-openshift you can not change the authetication method after Install !! If you installed the cluster with htpasswd, then change to LDAP the playbook trys to add a second authentication methot for the config. It is forbidden to add a second type of identity provider in the version 3.11 of Ansible-openshift. To solve this problem we must change the configuration manually.

### Environment
```
192.168.1.40    deployer
192.168.1.41    openshift01 # master node
192.168.1.42    openshift02 # infra node
192.168.1.43    openshift03 # worker node
```

### Configuration Gitlab
Login to Gitlab and create client for the app:
![Example image](/img/include/openshift_gitlab_sso.webp)

### Configurate The cluster
```
# on all openshift hosts
nano /etc/origin/master/master-config.yaml
...
  identityProviders:
  - name: gitlabsso
    challenge: true
    login: true
    mappingMethod: claim
    provider:
      apiVersion: v1
      kind: GitLabIdentityProvider
      legacy: true
      clientID: 7305abce637a123654a2c9dd4f8caec1156a1bc41cd80be4db0f14253fe24e58
      clientSecret: 2d5aebe7831c99383d876cc235febb401906263de748a29b03b058f62f15c2f7
      url: https://gitlab.devopstales.intra/
  - challenge: true
```

### Reconfigurate the cluster
```
# on all openshift hosts
master-restart api
master-restart controllers
```

