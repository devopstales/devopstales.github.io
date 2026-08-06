---
title: OKD OpenShift 4: Service Serving Certificate
url: https://devopstales.github.io/kubernetes/openshift4-service-serving-certificate/
date: 2023-02-25
keywords: openshift 4.11, openshift okd4, openshift container platform, rad hat openshift, buildconfig
---



In this Post I will show you how you can use Service Serving Certificate on OpenShift4.
<!--more-->

### What is Service Serving Certificate?

Service serving certificates are intended to support complex middleware applications that require encryption. Openshift automaticle generate a `x509.SHA256WithRSA` signature CA certificate stored in `service-ca` secret. These certificates are issued as TLS web server certificates. The service CA certificate, which issues the service certificates, is valid for 26 months and is automatically rotated when there is less than 13 months validity left. The generated certificate is only valid for the internal service DNS name `<service.name>.<service.namespace>.svc`, and is only valid for internal communications.

### OpenShift Service CA Operator

The openshift-service-ca-operator is an OpenShift Cluster Operator that runs in the cluster by default:

```bash
$  oc get co service-ca
NAME         VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
service-ca   4.4.12    True        False         False      4d4h
```

The Service CA Operator runs the operator in the openshift-service-ca-operator namespace:

```bash
$ oc get pod
NAME                                   READY   STATUS    RESTARTS   AGE
service-ca-operator-5f596775f8-gwlpj   1/1     Running   1          2d7h
```

The OpenShift Service CA Operator runs the following OpenShift controllers:

```bash
$ oc get pod -n openshift-service-ca
NAME                                            READY   STATUS    RESTARTS   AGE
apiservice-cabundle-injector-64cfdd7645-h4dqf   1/1     Running   3          2d7h
configmap-cabundle-injector-67ffc677d5-prn7b    1/1     Running   3          2d7h
service-serving-cert-signer-b5665b6f5-lx7bl     1/1     Running   3          2d7h
```

For OpenShift 4.x the service ca certificates are managed by the service CA operator.

The key and certificate are in a secret in the namespace openshift-service-ca as signing-key secret:

```bash
$ oc get secrets -n openshift-service-ca signing-key -o "jsonpath={.data['tls\.crt']}" |  base64 -d | openssl x509 -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1181511372096086528 (0x106592553fdf2200)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = openshift-service-serving-signer@1596638040
        Validity
            Not Before: Aug  5 14:34:00 2020 GMT
            Not After : Oct  4 14:34:01 2022 GMT
        Subject: CN = openshift-service-serving-signer@1596638040
...
```

### How to use Service Serving Certificate?
To activate Service Serving Certificate you need to annotate the service with `service.beta.openshift.io/serving-cert-secret-name`:

```bash
$ oc annotate service service-serving-cert service.beta.openshift.io/serving-cert-secret-name=service-serving-cert
```

Example Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-serving-cert
  annotations:
    service.alpha.openshift.io/serving-cert-secret-name: service-serving-cert
spec:
  ports:
  - name: service-serving-cert
    port: 443
    targetPort: 8443
  selector:
    app: service-serving-cert

```

Check certificate:

```bash
$ oc get secret service-serving-cert -o json | jq -r '.data."tls.crt"' | base64 --decode > service-serving-cert.pem
$ openssl crl2pkcs7 -nocrl -certfile service-serving-cert.pem | openssl pkcs7 -print_certs  -noout

subject=/CN=service-serving-cert.rbo.svc
issuer=/CN=openshift-service-serving-signer@1545507973

subject=/CN=openshift-service-serving-signer@1545507973
issuer=/CN=openshift-service-serving-signer@1545507973
```

Now the communication between the pods (truth a service as a proxy) is encrypted by a **self signed certificate** managed by Openshift. The applications wont see the certificate as valid because it is self signed. So we need to add the root certificate (CA) as trusted in the app.
To do this you can generate and mount as configmap.

### Add the service CA bundle to a config map

```bash
$ oc create -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: service-trustbundle-ca
  annotations:
    service.beta.openshift.io/inject-cabundle: "true"
data: {}
EOF

$ oc get configmap/service-trustbundle-ca -o jsonpath="{.data.service-ca\.crt}"  | openssl x509 -noout -subject -issuer -dates
subject= /CN=openshift-service-serving-signer@1593524307
issuer= /CN=openshift-service-serving-signer@1593524307
notBefore=Jun 30 13:38:26 2020 GMT
notAfter=Aug 29 13:38:27 2022 GMT
```

### Mount the configMap in your client container

You need to map the configMap in your client container which is service-A so that service-A gets access to client certificate.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: "my-service-a"
spec:
  selector:
    matchLabels:
      app: service-a
  template:
    metadata:
      labels:
        app: service-a
    spec:
      volumes:
      - name: client-certs-config
        configMap:
          name: "service-trustbundle-ca"
      containers:
      - name: service-a-pod
        image: my-image-repo/my-image-name:my-image-tag
        volumeMounts:
          - mountPath: /etc/servicea/client-certs
            name: client-certs-config
            readOnly: true
```
