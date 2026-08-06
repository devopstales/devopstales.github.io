---
title: Central authentication with oauth2-proxy
url: https://devopstales.github.io/kubernetes/k8s-central-oauth/
date: 2021-11-05
keywords: Keycloak, Kubernetes, dasboard, oauth2, SSO, OIDC
---


In this post I will show you how to use one central OAuth2 Proxy for multiple services inside your Kubernetes Cluster.

<!--more-->

The default example on how to secure a service with Nginx and OAuth2 Proxy shows you how to secure only one service. With this setup you need to create one oauth2-proxy for every service. . Another problem of this setup is that it is not supported by most Helm charts. 


First we need an oauth2-proxy to authenticate all of the requests:

```bash
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: auth-proxy
  namespace: ingress-system
spec:
  repo: "https://oauth2-proxy.github.io/manifests"
  chart: oauth2-proxy
  targetNamespace: ingress-system
  valuesContent: |-
    config:
      clientID: "<client-id>"
      clientSecret: "<clinet-secret>"
      cookieSecret: "bkJnQW1ua2xGa2tCV2pGTlZDdHJWS0t4SWJ2MFFSOWY="
    extraArgs:
      email-domain: '*'
      provider: keycloak
      login-url: https://<keycloak-url>/auth/realms/<keycloak-relm>/protocol/openid-connect/auth
      redeem-url: https://<keycloak-url>/auth/realms/<keycloak-relm>/protocol/openid-connect/token
      profile-url: https://<keycloak-url>/auth/realms/<keycloak-relm>/protocol/openid-connect/userinfo
      validate-url: https://<keycloak-url>/auth/realms/<keycloak-relm>/protocol/openid-connect/userinfo
      scope: email
      skip-provider-button: 'true'
      force-https: 'false'
      cookie-secure: 'false'
      pass-authorization-header: 'true'
      pass-basic-auth: 'false'
      skip-jwt-bearer-tokens: 'true'
      whitelist-domain: .k8s.intra
      cookie-domain: .k8s.intra
      oidc-issuer-url: https://<keycloak-url>/auth/realms/<keycloak-relm>
    ingress:
      enabled: true
      path: /oauth2
      hosts:
        - oauth.k8s.intra
      annotations:
        kubernetes.io/ingress.class: nginx
        cert-manager.io/cluster-issuer: ca-issuer
        nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"
      tls:
      - secretName: oauth2-proxy-tls-cert
        hosts:
          - oauth.k8s.intra
    metrics:
      enabled: true
#      servicemonitor:
#        enabled: true
#        namespace: "ingress-system"
```

For the service you want to secure, add the below annotations to the Ingress:

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: ca-issuer
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/auth-url: "http://auth-proxy-oauth2-proxy.ingress-system.svc.cluster.local/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://oauth.k8s.intra/oauth2/start?rd=https%3A%2F%2F$host$request_uri"
  name: alertmanager
spec:
  rules:
  - host: alertmanager.example.com
    http:
      paths:
      - backend:
          serviceName: alertmanager
          servicePort: 9093
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - alertmanager.example.com
    secretName: tls-cert
```

The `auth-sigin` redirects any needed login to the OAuth2 Proxy Ingress.
The `auth-url` annotation can access the OAuth2 Proxy internally via its service to verify a submitted token.

The OAuth2 Proxy will handle the authentication and later redirect you to the protected service again.
