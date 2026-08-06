---
title: kubedash 1.0
url: https://devopstales.github.io/kubernetes/kubedash-1.0/
date: 2023-03-20
keywords: Kubernetes, kubedash, kubernetes dashboard, admin dashboard, web ui, kubernetes user management, Image scan, image security
---


Today I am happy to announce the release of KubeDash 1.0. This blog post focuses on the functionality provided by the KubeDash 1.0. 

<!--more-->

## What is KubeDash?

KubeDash is a general purpose, web-based UI for Kubernetes clusters. It allows users to observe applications running in the cluster and troubleshoot them, as well as manage the cluster itself. It supports [OpenID Connect Tokens](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens) as a way to identify users who access the cluster

![KubeDash](/img/include/KubeDash_1.0_pic_01.webp)

The Default user is admin / admin

![KubeDash](/img/include/KubeDash_1.0_pic_03.webp)

![KubeDash](/img/include/KubeDash_1.0_pic_05.webp)

![KubeDash](/img/include/KubeDash_1.0_pic_04.webp)

You can watch the installed helm charts:

![KubeDash](/img/include/KubeDash_1.0_pic_02.webp)

## Usage

To ease deployment I created a helm chart for trivy-operator.

```bash
helm repo add devopstales https://devopstales.github.io/helm-charts
helm repo update
helm upgrade --install kubedash devopstales/kubedash
```

Create a value file for deploy:

```yaml
TimeZone: "CET"
logLevel: "INFO"
flaskConfig: "production" #or development

serviceAccount:
  create: true
  name: "kubedash-admin"

image:
  repository: devopstales/kubedash
  tag: 0.1
  pullPolicy: Always

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 10001

containerSecurityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["all"]

ingress:
  enabled: true
  url: "kubedash.mydomain.intra"
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    kubernetes.io/ingress.class: nginx
  tls:
    enabled: true
    tlsSecret: "mycert-tls"
    certManager:
      enabled: false
      clusterIssuer: "letsencrypt"
  whitelist:
    enabled: false
    ips: []

route:
  enabled: false
```

The following tables lists configurable parameters of the kubedash chart and their default values.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| TimeZone | string | `"CET"` | Time Zone in container |
| affinity | object | `{}` | Set the affinity for the pod. |
| containerSecurityContext | object | `{}` | list of the container's SecurityContexts |
| flaskConfig | string | `"production"` | flask environment: production or development |
| image.pullPolicy | string | `"Always"` | The docker image pull policy |
| image.repository | string | `"devopstales/kubedash"` | The docker image repository to use |
| image.tag | string | `"0.1-devel"` | The docker image tag to use |
| ingress.annotations | object | `{}` | Extra annotation to the Ingress object |
| ingress.enabled | bool | `true` | Enable Ingress object creation |
| ingress.tls.certManager.clusterIssuer | string | `"letsencrypt"` | Name of the certManager cluster issuer to use |
| ingress.tls.certManager.enabled | bool | `false` | Enable certManager |
| ingress.tls.enabled | bool | `true` | Enable tls on Ingress object |
| ingress.tls.tlsSecret | string | `""` | Name of the secret storing tls cert |
| ingress.url | string | `"kubedash.mydomain.intra"` | URL of the Ingress object |
| ingress.whitelist.enabled | bool | `false` | Enable ip blocking on ingress |
| ingress.whitelist.ips | list | `[]` | List of ips to allow communication |
| logLevel | string | `"INFO"` | Log level |
| nodeSelector | object | `{}` | Set nodeSelector for the pod |
| persistence.accessMode | string | `"ReadWriteOnce"` | Volumes mode |
| persistence.annotations | object | `{}` | Volumes annotations |
| persistence.enabled | bool | `true` | Volumes for the pod |
| persistence.size | string | `"1Gi"` | Volumes size |
| podSecurityContext | object | `{}` | list of the pos's SecurityContexts |
| route.annotations | object | `{}` | Extra annotation to the OpenShift Route object |
| route.enabled | bool | `false` | Enable OpenShift Route object creation |
| route.url | string | `"kubedash.mydomain.intra"` | URL of the OpenShift Route object |
| serviceAccount.create | bool | `true` | Enable automatic serviceAccount creation |
| serviceAccount.name | string | `"kubedash-admin"` | Configure the name of the serviceAccount |
| tolerations | list | `[]` | Set tolerations for the pod |

