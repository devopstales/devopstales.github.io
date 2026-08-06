---
title: Openshift SSO authentication
url: https://devopstales.github.io/sso/openshift-sso/
date: 2019-04-13
keywords: Keycloak, oauth2, SSO, OIDC, openshift
---


Configure Openshift Cluster to use Keycloak as a user backend for login with oauth2 and SSO.

<!--more-->

{{< content "/filedir/openshift.html" >}}

With Ansible-openshift you can not change the authetication method after Install !! If you installed the cluster with htpasswd, then change to LDAP the playbook trys to add a second authentication methot for the config. It is forbidden to add a second type of identity provider in the version 3.11 of Ansible-openshift. To solv this problem we must change the configuration manually.

### Environment
```
192.168.1.40    deployer
192.168.1.41    openshift01 # master node
192.168.1.42    openshift02 # infra node
192.168.1.43    openshift03 # worker node
```

### Configuration on Keycloak
```
create new client on keycloak in relm mydomain
Client ID: openshift
Clyent Protocol: openid-connect
Access type: confidential
Valid Redirect URIs: https://master.openshift.mydomain.itra/*
delete other urls

# On Credentials tap copy the secret to clientSecrethez in config.
```

### Configurate The cluster
```
# on all openshift hosts
nano /etc/origin/master/master-config.yaml
...
  identityProviders:
  - name: keycloak
    challenge: false
    login: true
    provider:
      apiVersion: v1
      kind: OpenIDIdentityProvider
      clientID: openshift
      clientSecret: ef03ffe6-854a-48b4-a26d-190c2861e3c8
      claims:
        id:
        - sub
        preferredUsername:
        - preferred_username
        name:
        - name
        email:
        - email
      urls:
        authorize: https://sso.devopstales.intra/auth/realms/mydomain/protocol/openid-connect/auth
        token: https://sso.devopstales.intra/auth/realms/mydomain/protocol/openid-connect/token
        logoutURL: "https://sso.devopstales.intra/auth/realms/mydomain/protocol/openid-connect/logout?redirect_uri=https://master.openshift.mydomain.itra/console"
  - challenge: true
```

### Reconfigurate the cluster
```
# on all openshift hosts
master-restart api
master-restart controllers
```

