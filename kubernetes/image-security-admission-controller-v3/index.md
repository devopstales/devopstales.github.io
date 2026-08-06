---
title: RKE2 Image security Admission Controller V3
url: https://devopstales.github.io/kubernetes/image-security-admission-controller-v3/
date: 2021-06-21
keywords: kubernetes deployment, kubernetes vs docker, Kubernetes Secuity, best Practices, rancher, Admission Controller, trivy
---


In a previous posts we talked about the [anchore-image-validator made by Banzaicloud](https://devopstales.github.io/home/image-security-admission-controller/) and the [admission-controller made by Anchore](https://devopstales.github.io/home/image-security-admission-controller-v2/). In this post I will show you my own admission-controller for image scanning.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

I found multiple solution for Anchore Engine but only one for Trivy. The [trivy-enforcer](https://github.com/aquasecurity/trivy-enforcer) that is an experimental project and use OPA for enforce the policy. So I decide to create mey own dmission-controller.

### How an admission controller works
An Admission Controller Webhook is triggered when a Kubernetes object is created. It sends a JSON formatted HTTP request to a specific Kubernetes Service in a namespace which returns a JSON response. If you whoud like to now more aboute admission controllers you can read about it in my previous post [Using Admission Controllers](https://devopstales.github.io/kubernetes/admission-controllers/) 

### Writing a Validating Admission Controller

I want to walidate the Pod object to check how many vulnerability has the image in this pod. So I wrote a [python script](https://github.com/devopstales/trivy-image-validator/blob/master/trivy-scanner.py) that will pars the JSON request for the pods image, rin a trivy scan on it and sen back the answer. Then I build it to a docker image called `devopstales/trivy-scanner-admission:1.0.1`. I run it as a deployment:

```yaml
--- 
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
  name: trivy-cache
spec: 
  accessModes: 
    - ReadWriteOnce
  resources: 
    requests: 
      storage: 1G
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trivy-scanner
  labels:
    app: trivy-scanner
  namespace: validation
spec:
  replicas: 1
  selector:
    matchLabels:
      app: trivy-scanner
  template:
    metadata:
      labels:
        app: trivy-scanner
    spec:
      securityContext:
        fsGroup: 10001
      containers:
        - name: trivy-scanner
          image: devopstales/trivy-scanner-admission:1.0.1
          imagePullPolicy: Always
          volumeMounts:
          - name: cache
            mountPath: "/home/kube-trivy-admission/.cache/trivy"
#          - name: config-json
#            mountPath: "/home/kube-trivy-admission/.docker"
      volumes:
      - name: cache
        persistentVolumeClaim:
            claimName: "trivy-cache"
#      - name: config-json
#        secret:
#          secretName: config-json
---
apiVersion: v1
kind: Service
metadata:
  name: trivy-scanner
  namespace: validation
spec:
  selector:
    app: trivy-scanner
  ports:
  - port: 443
    targetPort: 5000
```

The Service must be an HTTPS port on 443 for the Admission Webhook so I created a self-signed certificate and placed in the docker container.

### Create the Admission Webhook

The abow Admission Webhook will send teh HTTP request to my `trivy-scanner` service:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: trivy-scanner
webhooks:
  - name: trivy-scanner.devopstales.intra
    sideEffects: "None"
    admissionReviewVersions: [v1beta1, v1]
    clientConfig:
      service:
        name: trivy-scanner
        namespace: validation
        path: "/validate"
      caBundle: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUZMekNDQXhlZ0F3SUJBZ0lVWnBZdlRuUUFWRTgvZk9jMHJWeFhWU1hadTBnd0RRWUpLb1pJaHZjTkFRRUwKQlFBd0p6RWxNQ01HQTFVRUF3d2NRV1J0YVhOemFXOXVJRU52Ym5SeWIyeHNaWElnVjJWaWFHOXZhekFlRncweQpNVEExTXpBeE5URTJORE5hRncweU1UQTJNamt4TlRFMk5ETmFNQ2N4SlRBakJnTlZCQU1NSEVGa2JXbHpjMmx2CmJpQkRiMjUwY205c2JHVnlJRmRsWW1odmIyc3dnZ0lpTUEwR0NTcUdTSWIzRFFFQkFRVUFBNElDRHdBd2dnSUsKQW9JQ0FRREhpZyt2Yjl2K3A1SldhYzNnUzhiNzJOZ1hFTkFFU1FYMHZWZTlGUzNhUURqRlJWcDhYVlEzZkdKYQp2SmFTdjVuNWtDVkRhZDkvcEtRaEZ5amI1OUJpRmVySVU1dW40c1BhRGRJUThtb25pL0VNbFZRNURNK0JNZzRoCnc0bk12N1BCNFRSbGFFSlNEVVBpaFZqNUlGRE9VbjFoMjVQMGpPSktUT2NWSW5HTXFpeldBenptZlhXb2pnK1UKMnl0VWhNYlBwS2M1TE5XS3p2cmhERUY2eVBXbHN1d0VPalhIOUhxQXQzQmxYVVYvOWV3aWdjRFFjVHk1Ty9WUAo0OEFVam9TTGlIR254QzI5S21qVDhwaHhOUzV5emhXbkxIUkdkWTc0UmZDTlZ4akpqRk8zMlo1TjFMRGJsRTdiCmlSU3ovUHlvcVZFb3NIcmZXV3QrbUhzN21ZODNGc3lhYnFjeklyaGdyR0Z5TSs1MWtJZ1lUbmV5M3VkMHVXd0MKMVlWVHdLQ3Rsa3VMSWUzc29yV2V2THRLYlRuZXpzUWFST0lndmY4dTBjQVpLTzljMFN3SXN2RjZPK3RKSXQ4RQo1MjRMUWhQeDhEejVSVktDUGgwaGxZR3c0VmN5ZFdZNTNGVFBPYmtIaHpVelErTXlBcDZSN1NVbHhIdW1HTVFTCkd5ZzlDZVJFSzA3MWs5bmNpUWcvb3FZQWpxay8rUDUvT0c0ZDFyQjA0cmFzRTdRcXk5YUQvTStLOGFTOWhWVWQKYTlFR2NKdmZyUGtwSTZ6MDhFdU5sSHN0Y0NpMUtyb0VzTXlXSlFuSEhZWXAvckJPdEpSeVBueGZIY05vc3diWgorYnZBbVdlRDE1Wm1NNW1aT01vSlFqY1BnMWdDZXdlRkZiWi9rN0xadTJvTHRjNVU3UUlEQVFBQm8xTXdVVEFkCkJnTlZIUTRFRmdRVTVtOVozQnZjUTRqSEUvSkVRRXBMeUN1ay9pWXdId1lEVlIwakJCZ3dGb0FVNW05WjNCdmMKUTRqSEUvSkVRRXBMeUN1ay9pWXdEd1lEVlIwVEFRSC9CQVV3QXdFQi96QU5CZ2txaGtpRzl3MEJBUXNGQUFPQwpBZ0VBVWo1NEpDNldCb2JWQ0cwRzQwc0ltNktvZEVQWXZLTnlhSUNFb25HSlFZTlpKYndUdGR5T1NlbXhzdENzCkdHM2h6bk5SUCtCME43ZUhlR2JQZENqcjNzSElSTTYxQmk0UDVoQ1BGSW9YdDgvdWJrSzQyR2dpd3ZNK1I4NlgKWUlTY2FJS3A5OXUyQjBsM3c3Q3pNSDZFcmR0aGM2S2RZY2dCWXhDaVBKR2trQ2wwelUwU1ZuYTVkbTExNlBMTAppL05LbUZjbitBbDl4TThqQmhyZU5mWGNITnVJNzFBSzluYnZzMkNLMkMrSUw2ZGpqaVVNTFdCNzRUZVBTNk1pCkpjblVleUQ1NGxiK2dWMTRZY0NKeGxSSkJQR3FpVFVTbE44cFdwQkpISlo5WmVjcFlHbWM5blNsK0tNZ3RFb0gKNHk3NS9ZZUhlYVo3UktGWDBOZWlpRC81NHFMM0Q3RTNmV25BclQ5ZUVjYi9ON1Z4MDdWczNlSWhheCtSNSt2QgpPcU1jaTVHSzY1NVUyeUpVMlR3SVNFSTFRZmd2TDNLZmxXU0c2WkUwSkg0bE1MZmJSMHg2SmtvajJzTTRYQks2CjE0NXh6eFdIZ2pVTzFqcnpmNVdRUi9MTXd0b3dVcFlBZWwrTWdMNnZBby9sbGw3THl2alNFL1Z6TEdNVFJyL2EKd1VJbFpYaHFreW5LeEJTUTk0Zi8vOTZLeWorQzk4WVQxcVFpVHU1aWQvYS82S2paWlZJVGFaYlRySk9zWnBHLwpRSGdpT3FFbDlWUGpCOUdtTUdhaklSbHJiRkp1R0FHQVlhalpvd2VVeWdaL3BocEd1NUh6dzJTaTRtaHUxT0tpCmVoR3diUzdoTHlvZ3hYelk4VTA1ZXBmcEJuTERFc09HWThjVkd0bVdFNk9HdGhvPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg=="
    rules:
      - apiGroups: ["apps", ""]
        resources:
          - "pods"
        apiVersions:
          - "*"
        operations:
          - CREATE
```

I placed the root CA of my self-signed certificate in this Validating Webhook Configuration to make my Service's certiface valid for the Kubernetes api.

### Policy

Now If I create a Deployment, Pod ... It scan the Image with tryvi. To block an Image I need to add the limits of the maximum numer of vulnerability for the severities.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      annotations:
        trivy.security.devopstales.io/medium: "5"
        trivy.security.devopstales.io/low: "10"
        trivy.security.devopstales.io/critical: "2"
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - name: http
          containerPort: 80
```

