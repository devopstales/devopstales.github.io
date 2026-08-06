---
title: kube-openid-connect 1.0
url: https://devopstales.github.io/kubernetes/kube-openid-connect-1.0/
date: 2022-03-25
keywords: Kubernetes, authentication, kubectl, kubectl plugin, oauth2, oidc, openid
---


Today I am happy to announce the release of kube-openid-connect 1.0 and assign the first ever stable release number. This blog post focuses on the functionality provided by the kube-openid-connect 1.0 release. 

<!--more-->

### What is kube-openid-connect?

Kube OpenID Connect is an application that can be used to easily enable authentication flows via OIDC for a kubernetes cluster. Kubernetes supports [OpenID Connect Tokens](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens) as a way to identify users who access the cluster. Kube OpenID Connect helps users with it's `kubectl` plugin to authenticate and get `kubectl` config.

### How It Works

Kube OpenID Connect has two main component the server an the `kubectl plugin`. The server is written in python and the `kubectl plugin` in go for easier multi architecture build. When you want to login to a Kubernetes cluster you just simply use the `kubectl login` command to connect to the server component. It will open the OpenID authentication page in you browser. After you successfully logged in the server based on yout `JWT token` generates a `kubectl config` and push back to your `kubectl plugin`, that writes it to your config.

### OIDC (Provider) Setup

You will need to obtain the OIDC details of the provider you need to use. This will contain the Issuer URL, Client ID and the Client Secret.
In the case of Google (The provider which was used when initially creating this) go to the [Developer / Credentials](https://console.developers.google.com/apis/credentials) console. You will need to add the ingress url to both
* *Authorised JavaScript origins* - https://kubeauth.k8s.intra
* *Authorised redirect URIs* - https://kubeauth.k8s.intra/callback

If you used kops the credentials you're after are
```yaml
apiVersion: kops/v1alpha2
kind: Cluster
  authorization:
    rbac: {}
  kubeAPIServer:
    authorizationRbacSuperUser: admin
    oidcClientID: UNIQUE_ID_REDACTED.apps.googleusercontent.com
    oidcIssuerURL: https://accounts.google.com
    oidcUsernameClaim: email
```

For G Suite :
The redacted part of a ClientID is about 45 alphanumeric characters long (may also contain a hyphen or two)
The client secret will be about 25 alphanumeric chacters (may also contain a hyphen or two)

For manually configure the Kubernetes ApiServer integration with OpeniD provider:

```yaml
nano /etc/kubernetes/manifests/kube-apiserver.yaml
---
...
command:
- /hyperkube
- apiserver
...
- --oidc-issuer-url={{ .Values.server.oidcServerURL }}
- --oidc-client-id={{ .Values.server.oidcClientID }}
- --oidc-username-claim=email
- --oidc-groups-claim=groups
# for self sign cert or custom ca
#- --oidc-ca-file=/etc/kubernetes/pki/rootca.pem
```

```bash
systemctl restart kubelet
```


## Install the server
To ease deployment I created a helm chart for kube-openid-connect.

```bash
helm repo add devopstales https://devopstales.github.io/helm-charts
helm repo update
```

Create a value file for deploy:

```yaml
cat <<'EOF'> values.yaml
# Default values for kube-openid-connect.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

TimeZone: Europe/Budapest

server:
  debug: false
  oidcRedirectUrlHttpScema: https
  # redirect after logout
  oidcOutURL: https://devopstales.github.io/tags/kube-openid-connect/
  # http or https
  oidcRedirectUrlHost: kubeauth.k8s.intra
  # same es ingres host
  oidcServerURL: https://sso.k8s.intra/auth/realms/homelab
  oidcClientID: k8s-auth
  oidcSecret: ashdkhsadhasdhakjshdakshdash
  k8sContext: k8s-cl01
  k8sApiServer: https://192.168.0.10:6443
  # `k8sApiServer` is the url for kubectl
  #   This is typically  https://api.fqdn
  k8sCaCrt: |-
    -----BEGIN CERTIFICATE-----
    CA
    -----END CERTIFICATE-----
# `caCrt` is the public / CA cert for the cluster
# cat /etc/kubernetes/pki/apiserver.crt

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: ca-issuer
    nginx.ingress.kubernetes.io/proxy-buffer-size: "64k"
    ingress.kubernetes.io/force-ssl-redirect: "true"
  hosts:
    - host: kubeauth.k8s.intra
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls:
    - secretName: kubeauth-tls
      hosts:
        - kubeauth.k8s.intra

EOF
```

The following tables lists configurable parameters of the trivy-operator chart and their default values.

|               Parameter             |                Description                  |                  Default                 |
| ----------------------------------- | ------------------------------------------- | -----------------------------------------|
| TimeZone                            | Tomezone for container time | Europe/Budapest |
| server.debug                        | enable debug logging | false |
| server.oidcOutURL                   | Where to redirect after logout? | https://devopstales.github.io/tags/kube-openid-connect/ |
| server.oidcRedirectUrlHttpScema     | Ingress schema (http or https) | http |
| server.oidcRedirectUrlHost          | Ingress hostname | chart-example.local |
| server.oidcServerURL                | URL of OIDC provider endpoint | Not set |
| server.oidcClientID                 | Your unique client ID | Not set |
| server.oidcSecret                   | The password for the Client ID | Not set |
| server.k8sContext                   | Context of the cluster in generated config | Not set |
| server.k8sApiServer                 | The endpoint for kubectl to use | Not set |
| server.k8sCaCrt                     | The Public CA cert for the cluster | Not set |
| image.repository                    | image | devopstales/trivy-operator | See values.yaml |
| image.pullPolicy                    | pullPolicy | Always |
| image.tag                           | image tag | 1.0 |
| imagePullSecrets                    | imagePullSecrets list | `[]` |
| nameOverride                        | Override the name part | Not set |
| fullnameOverride                    | Override the full name | Not set |
| serviceAccount.create               | create serviceAccount | true |
| serviceAccount.annotations          | add annotation to serviceAccount | `{}` |
| serviceAccount.name                 | name of the serviceAccount | Not set |
| podAnnotations                      | add extra annotations for the pod | `{}` |
| podSecurityContext.fsGroup          | mount id | 10001 |
| securityContext                     | add extra securityContext for the pod | `{}` |
| service.port                        | port number | 5000 |
| service.type                        | type of the service | ClusterIP |
| ingress.enabled                     | enable ingress | true |
| ingress.className                   | ingress calss | Not Set |
| ingress.annotations                 | extra annotation to ingress | Not set |
| ingress.hosts.host                  | hostname for ingress | chart-example.local |
| ingress.hosts.paths[0].path         | subpath on ingress | `\` |
| ingress.hosts.paths[0].pathType     | ingress path type | ImplementationSpecific |
| ingress.tls                         | tls config for ingress | `[]` |

```bash
kubectl create ns kubeauth
kubens kubeauth
helm upgrade --install kubelogin devopstales/kube-openid-connect -f values.yaml
```

### Install the Kubectl plugin

```bash
# Homebrew (macOS and Linux)
brew tap devopstales/devopstales
brew install kubectl-login

# Krew (macOS, Linux, Windows and ARM)
kubectl krew install openid-connect

# Chocolatey (Windows)
choco install kubectl-login

# Binary release (Windows, macOS and Linux)
https://github.com/devopstales/kube-openid-connect/releases
```

### Use the plugin to login:

Point the url to the ingress of the server component:

```bash
$ kubectl login https://kubeauth.k8s.intra
Configfile created with config for productioncluster to ~/.kube/config
Happy Kubernetes interaction!
```
