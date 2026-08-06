---
title: Configure OKD OpenShift 4 authentication
url: https://devopstales.github.io/kubernetes/openshift4-auth/
date: 2021-12-13
keywords: openshift 4.2, openshift okd4, openshift container platform, rad hat openshift
---



In this Post I will show you how you can create multiple ingress route on an OpenShift 4 on premise.
<!--more-->

{{< content "/filedir/openshift4.html" >}}

### Create a Cluster Admin User

The current kubeadmin user that we are using in the previous step is temporary. We need to create a permanent cluster administrator user. The fastest way to do this is by using htpasswd as an authentication provider. We will create a secret under the openshift-config namespace and add a htpasswd provider in the cluster oAuth.

```bash
htpasswd -c -B -b users.htpasswd adminuser adminpassword
htpasswd -c -B -b users.htpasswd testuser testpassword
oc create secret generic htpass-secret --from-file=htpasswd=users.htpasswd -n \
openshift-config
oc apply -f htpasswd_provider.yaml
```

You can create groups to the cluster:

```yaml
nano groups.yaml
---
kind: Group
apiVersion: user.openshift.io/v1
metadata:
  name: admins
users:
  - adminuser
---
kind: Group
apiVersion: user.openshift.io/v1
metadata:
  name: cluster-admins
users:
  - adminuser
---
apiVersion: user.openshift.io/v1
kind: Group
metadata:
  name: developers
users:
  - testuser
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: okd-admins
subjects:
  - kind: Group
    apiGroup: rbac.authorization.k8s.io
    name: admins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: okd-cluster-admin
subjects:
  - kind: Group
    apiGroup: rbac.authorization.k8s.io
    name: cluster-admins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: okd-cluster-developers
subjects:
  - kind: Group
    apiGroup: rbac.authorization.k8s.io
    name: developers
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: basic-user
```

```bash
oc apply -f group-admins.yaml
```

### Enable Oauth On OKD Cluster

I will use Keycloak as the oauth prowider. In Keycloak `okd` realm I created a client called `okd4`. The `openid-client-secret` is the base64 encodid secret for the `okd4` Client ID.

```yaml
nano oauth-config.yaml
---
kind: Secret
apiVersion: v1
metadata:
  name: openid-client-secret
  namespace: openshift-config
data:
  clientSecret: OWI4OWUyZjgtYrM6ZC10ODU2LTgyN3YtN2ZiODUzNDUyZDc4
type: Opaque
---
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - mappingMethod: add
      name: sso.mydomain.intra
      openID:
        claims:
          email:
            - email
          name:
            - name
          preferredUsername:
            - email
            - preferred_username
        clientID: okd4
        clientSecret:
          name: openid-client-secret
        extraScopes: []
        issuer: 'https://sso.mydomain.intra/auth/realms/okd'
      type: OpenID
```

Then apply the config `oc apply -f oauth-config.yaml`

### Fix: x509: certificate signed by unknown authority

If you get the fallowing error: The authentication operator can't honor OAuth configuration due to an `x509: certificate signed by unknown authority` error 

Check the `openshift-authentication-operator` pod log:

```bash
oc -n openshift-authentication-operator logs $(oc -n openshift-authentication-operator get pods -l app=authentication-operator -o=custom-columns=NAME:.metadata.name --no-headers)
[...]
E1125 15:31:27.093873       1 oauth.go:69] failed to honor IDP v1.IdentityProvider{Name:"sso", MappingMethod:"claim", IdentityProviderConfig:v1.IdentityProviderConfig{Type:"OpenID", BasicAuth:(*v1.BasicAuthIdentityProvider)(nil), GitHub:(*v1.GitHubIdentityProvider)(nil), GitLab:(*v1.GitLabIdentityProvider)(nil), Google:(*v1.GoogleIdentityProvider)(nil), HTPasswd:(*v1.HTPasswdIdentityProvider)(nil), Keystone:(*v1.KeystoneIdentityProvider)(nil), LDAP:(*v1.LDAPIdentityProvider)(nil), OpenID:(*v1.OpenIDIdentityProvider)(0xc010181ef0), RequestHeader:(*v1.RequestHeaderIdentityProvider)(nil)}}: x509: certificate signed by unknown authority
I1125 15:31:28.369400       1 status_controller.go:165] clusteroperator/authentication diff {"status":{"conditions":[{"lastTransitionTime":"2019-11-20T10:17:18Z","message":"IdentityProviderConfigDegraded: failed to apply IDP sso config: x509: certificate signed by unknown authority","reason":"AsExpected","status":"False","type":"Degraded"},{"lastTransitionTime":"2019-11-22T11:41:09Z","reason":"AsExpected","status":"False","type":"Progressing"},{"lastTransitionTime":"2019-10-26T16:15:59Z","reason":"AsExpected","status":"True","type":"Available"},{"lastTransitionTime":"2019-10-26T13:30:53Z","reason":"AsExpected","status":"True","type":"Upgradeable"}]}}
```

```yaml
nano oauth-config.yaml
---
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - mappingMethod: add
      name: sso.mydomain.intra
      openID:
        ca:
          name: sso-ca-config-map
...
---
apiVersion: v1
data:
  ca.crt: |+
    -----BEGIN CERTIFICATE-----
    MIIEFTCCAv2gAwIBAgIGSUEs...
    -----END CERTIFICATE-----

kind: ConfigMap
metadata:
  name: sso-ca-config-map
  namespace: openshift-config
```

