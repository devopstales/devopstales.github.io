---
title: Microk8s: Unable to connect to the server: x509: certificate has expired or is not yet valid
url: https://devopstales.github.io/kubernetes/microk8s-expired-cert/
date: 2023-01-10
keywords: Microk8s, certificate, expired, x509
---


In this Post I will shoe you how to renew the kubernetes api cert in Microk8s.

<!--more-->

```bash
# microk8s kubectl get all --all-namespaces
Unable to connect to the server: x509: certificate has expired or is not yet valid: current time 2023-01-11T10:50:25+01:00 is after 2022-07-21T13:27:27Z
```

So the k8s api certificate probably expired. Test it:

```bash
# sudo microk8s.refresh-certs -c
The CA certificate will expire in 0 days.
```

Ok so we renew the certificate:

```bash
# microk8s.refresh-certs
Taking a backup of the current certificates under /var/snap/microk8s/2530/var/log/ca-backup/
Creating new certificates
Signature ok
subject=/C=GB/ST=Canonical/L=Canonical/O=Canonical/OU=Canonical/CN=127.0.0.1
Getting CA Private Key
Signature ok
subject=/CN=front-proxy-client
Getting CA Private Key
1
Creating new kubeconfig file
2023-01-11T10:53:03+01:00 INFO Waiting for "snap.microk8s.daemon-cluster-agent.service" to stop.
Stopped.
Started.

The CA certificates have been replaced. Kubernetes will restart the pods of your workloads.
Any worker nodes you may have in your cluster need to be removed and re-joined to become aware of the new CA.
```

