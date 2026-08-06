---
title: Secure your applications with Pomerium Ingress Controller
url: https://devopstales.github.io/kubernetes/k8s-pomerium-ingress-controller/
date: 2022-06-14
keywords: kubernetes deployment, ingress controller, oauth proxy, oauth2 proxy, pomerium, oidc, OpenID
---


In this blog post, I will show you how you can install Pomerium Ingress Controller and use it to secure your application.

<!--more-->

I a [previous post](/kubernetes/k8s-central-oauth/) I showed how you can authenticate users for specific ingresses by oauth2-proxy and nginx ingress controller. That solution works grate, but it has a flow. If you want not just authenticate but authorize users based on groups, you need multiple oauth2-proxy. That is not ideal. But wit Pomerium Ingress Controller you can configure authentication and authorization by annotations on the ingress.

Pomerium use certificate-manager to generates certificates for secure connections. So first I will install that:

```bash
kubectl create ns cert-manager
kubens cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --namespace cert-manager \
--create-namespace --set installCRDs=true
```

To install Pomerium Ingress Controller I will use it's helm chart:

```bash
kubectl create ns ingress-system
kubens ingress-system
helm repo add pomerium https://helm.pomerium.io
helm repo update
```

Now I will create the values file:

```yaml
nano values.yaml
---
authenticate:
  idp:
    provider: "oidc"
    clientID: "oauth2_proxy"
    clientSecret: "1e8ee274-513e-41c0-903f-b3cg13147dgb"
    url: "https://sso.k8s.intra/auth/realms/cl12"
    serviceAccount: "pomerium-authenticate"
  ingress:
    annotations:
      cert-manager.io/cluster-issuer: ca-issuer
    tls:
      secretName: authenticate.k8s.intra
redis:
  enabled: false
  generateTLS: true
ingressController:
  enabled: true
  ingressClassResource:
    enabled: true
    default: true
    name: "pomerium"
config:
  rootDomain: k8s.intra #Change this to your reserved domain space.
  generateTLS: true # On by default, disabled when cert-manager or another solution is in place.
ingress:
  enabled: false
```

```bash
helm upgrade --install pomerium pomerium/pomerium --values ./pomerium-values.yaml
```

To demonstrate the functionality of Pomerium Ingress Controller I will create a deo application:

```yaml
nano demo_app.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuard
  namespace: default
spec:
  selector:
    matchLabels:
      app: kuard
  replicas: 1
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-amd64:1
        imagePullPolicy: Always
        name: kuard
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: kuard
  namespace: default
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: kuard
```

Now I will show so a few Ingress examples for this app. I the firs example we wil allow the authentication for a specific user by its email.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.pomerium.io/policy: '[{"allow":{"and":[{"email":{"is":"user@mydomain.intra"}}]}}]' # This can also be a yaml block quote
spec:
  rules:
  - host: kuard.k8s.intra
    http:
      paths:
      - backend:
          service:
            name: kuard
            port:
              name: http
        path: /
        pathType: Prefix
```

As you can see you can add the policy as a yaml block quote.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.pomerium.io/policy: |
      - allow:
          and:
            - email:
                is: user@mydomain.intra
spec:
  rules:
  - host: kuard.k8s.intra
    http:
      paths:
      - backend:
          service:
            name: kuard
            port:
              name: http
        path: /
        pathType: Prefix
```

In the second example I will allow the authentication for all user from a specific domain.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.pomerium.io/policy: |
      - allow:
          and:
            - email:
                ends_with: "@mydomain.intra"
spec:
  rules:
  - host: kuard.k8s.intra
    http:
      paths:
      - backend:
          service:
            name: kuard
            port:
              name: http
        path: /
        pathType: Prefix
```

OR:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.pomerium.io/policy: |
      - allow:
          and:
            - domain:
                is: "mydomain.intra"
spec:
  rules:
  - host: kuard.k8s.intra
    http:
      paths:
      - backend:
          service:
            name: kuard
            port:
              name: http
        path: /
        pathType: Prefix
```

The last example shows you how you can use regex in your ingress.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/issuer: example-issuer
    ingress.pomerium.io/policy: |
      - allow:
          and:
            - domain:
                is: "mydomain.intra"
    ingress.pomerium.io/path_regex: "true"
  name: example
spec:
  ingressClassName: pomerium
  rules:
  - host: example.localhost.pomerium.io
    http:
      paths:
      - backend:
          service:
            name: example
            port:
              name: http
        path: ^/(admin|superuser)/.*$
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - example.localhost.pomerium.io
    secretName: example-tls
```

As you can see Pomerium is a very versatile Ingress controller. For more examples fo to [Pomerium's page](https://www.pomerium.com/docs/k8s/ingress.html).

