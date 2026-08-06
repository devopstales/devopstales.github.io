---
title: Kubectl authentication with Kuberos
url: https://devopstales.github.io/sso/k8s-kuberos/
date: 2020-01-07
keywords: Keycloak, kuberos, kubernetes, oauth2, SSO, OIDC
---


Kuberos is an OIDC authentication helper for kubectl login.

<!--more-->

> Kuberos hase bean dpricated at May 10, 2019.

{{< content "/filedir/kubernetes.html" >}}

```bash
nano /etc/kubernetes/manifests/kube-apiserver.yaml
...
    command:
    - /hyperkube
    - apiserver
    - --advertise-address=10.10.40.30
...

    - --oidc-issuer-url=https://keycloak.devopstales.intra/auth/realms/mydomain
    - --oidc-client-id=k8s
    - --oidc-username-claim=email
    - --oidc-groups-claim=groups
...

systemctl restart docker kubelet
```

```yaml
cat <<'EOF'> values.yaml
replicaCount: 1

kuberos:
  oidcClientURL: https://keycloak.devopstales.intra/auth/realms/mydomain
  oidcClientID: k8s
  oidcSecret: 43219919-0904-4338-bc0f-c986e1891a7a
  clusters:
  - name: openshift
    apiServer: https://192.168.0.106:6443
    # `apiServer` is the url for kubectl
    #   This is typically  https://api.fqdn
    caCrt: |-
      -----BEGIN CERTIFICATE-----
      MIIDZDCCAkygAwIBAgIIe/R9sc8oJiAwDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
      AxMKa3ViZXJuZXRlczAeFw0xOTEyMjcxNzM3MzlaFw0yMDEyMjYxNzM3MzlaMBkx
      FzAVBgNVBAMTDmt1YmUtYXBpc2VydmVyMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A
      MIIBCgKCAQEAzAg7MflA/HVTVcZPsGJH71cfcJ/U1CtEYbXfu/AQbhGg09XKmeK9
      aGEK3kSgi/Hyoi7M+e/ntx1+Gp/jwc8kanMFRLgxdKCxxi4MOswZF/q2loUdNoE/
      OQVPWQi8Hgznubw/0gINUkIq8mRx9Bb+RcRnJEfD3CXkxDhUNeCvvjeTrujguF0h
      pgfzrLoc2kGdJYpHiLqow8jRq7XXk0RzZaqCQjAEZgqWamwbTTqFZh3v+1gF/2s0
      EbFVVL2Ctu1dOGe1FkZxte7/Po1XBkPLQuRXbH3QRiJkPfyOW16T1nWk1QTcpCdH
      HO/l+CY2nLPFZL1BM83QuVmPgR1T1p+5tQIDAQABo4GzMIGwMA4GA1UdDwEB/wQE
      AwIFoDATBgNVHSUEDDAKBggrBgEFBQcDATCBiAYDVR0RBIGAMH6CEms4cy5teWRv
      bWFpbi5pbnRyYYIKa3ViZXJuZXRlc4ISa3ViZXJuZXRlcy5kZWZhdWx0ghZrdWJl
      cm5ldGVzLmRlZmF1bHQuc3ZjgiRrdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0
      ZXIubG9jYWyHBApgAAGHBMCoAGowDQYJKoZIhvcNAQELBQADggEBAJZ7jxPR72V6
      PYL3SKCWaS+RgTuGuSm0pYu26cBmPOjsugd8DUrJ7+iAnKDHUmmw22sWheLLCokc
      YU/AIfdbbsz0+f+/qthkO7zJmAJgdIAOMJ5MQCbxMBt+6L813r1R3QI7kAGxHvzV
      loKJVIIHq/6K3gFEZDfo0myvNvtOIpBCeMnZRK+8hx3UNcHckZbhkan1Z1j9t9iw
      b6Vv5jY1+9t2Iltd2wuNaUvHicx+3X6JPAqVR6H0jI3i+QSyT1EHXtBtbQBBpP4T
      5WDz+9uDa1mIDHtww7DTnJwY+hGI7fVF2H7XQaM4xwhGnwIwbkSh45JWVtUEHMou
      Q7T4bTyrwuQ=
      -----END CERTIFICATE-----
    # `caCrt` is the public / CA cert for the cluster
    # cat /etc/kubernetes/pki/apiserver.crt

ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-buffer-size: "64k"
    cert-manager.io/cluster-issuer: ca-issuer
    ingress.kubernetes.io/force-ssl-redirect: "true"
  path: /
  hosts:
    - kubectl.devopstales.intra
  tls:
    - secretName: default-cert
      hosts:
        - kubectl.devopstales.intra

image:
  repository: negz/kuberos
  tag: ede4085
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  annotations: {}
  # Add your service annotations here.

resources: {}
EOF
```

```bash
helm repo add devopstales https://devopstales.github.io/helm-charts
helm upgrade --install kuberos devopstales/kuberos --namespace kuberos -f values.yaml
```

![kuberos](/img/include/kuberos.webp)

Below is an example of a `ClusterRoleBinding` that binds the role `cluster-admin` to the Keycloak group `devops-team`. (In my case it came from ldap) Create your own role bindings to fit your needs and apply them to the cluster.

```yaml
nano devops-team_ClusterRoleBinding.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-it-afdeling
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: devops-team
```

```yaml
nano user_ClusterRoleBinding.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: devopstales-admin
subjects:
  - kind: User
    apiGroup: rbac.authorization.k8s.io
    name: devopstales
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

