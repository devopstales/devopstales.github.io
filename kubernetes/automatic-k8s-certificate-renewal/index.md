---
title: Automatic Kubernetes Certificate Renewal
url: https://devopstales.github.io/kubernetes/automatic-k8s-certificate-renewal/
date: 2024-12-20
keywords: kubernetes deployment, kubernetes vs docker, Kubernetes Secuity, best Practices
---


In this post I will show you how you can automate the Kubernetes Certificate renewal.
<!--more-->

Create a Bash script for renewing the certificates:

```bash
nano /usr/local/bin/k8s-certs-renew.sh

echo "## Expiration before renewal ##"
/usr/local/bin/kubeadm certs check-expiration

echo "## Renewing certificates managed by kubeadm ##"
/usr/local/bin/kubeadm certs renew all

echo "## Restarting control plane pods managed by kubeadm ##"
/usr/local/bin/crictl pods --namespace kube-system --name 'kube-scheduler-*|kube-controller-manager-*|kube-apiserver-*|etcd-*' -q | /usr/bin/xargs /usr/local/bin/crictl rmp -f

echo "## Updating /root/.kube/config ##"
cp /etc/kubernetes/admin.conf /root/.kube/config

echo "## Waiting for apiserver to be up again ##"
until printf "" 2>>/dev/null >>/dev/tcp/127.0.0.1/6443; do sleep 1; done

echo "## Expiration after renewal ##"
/usr/local/bin/kubeadm certs check-expiration
```

Create a systemd service to call the script:

```bash
/etc/systemd/system/k8s-certs-renew.service

[Unit]
Description=Renew K8S control plane certificates

[Service]
Type=oneshot
ExecStart=/usr/local/bin/k8s-certs-renew.sh
```

Create a systemd timer to trigger the service at regular intervals:

```bash
cat /etc/systemd/system/k8s-certs-renew.timer

[Unit]
Description=Timer to renew K8S control plane certificates

[Timer]
OnCalendar=Fri *-*-1,2,3,4,5,6,7 03:10:00
RandomizedDelaySec=30min
Persistent=yes

[Install]
WantedBy=multi-user.target
```

Execute the following commands to complete the process:

```bash
systemctl daemon-reload

systemctl restart k8s-certs-renew.timer
```

