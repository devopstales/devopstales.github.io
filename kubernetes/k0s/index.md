---
title: K0S The tiny Kubernetes
url: https://devopstales.github.io/kubernetes/k0s/
date: 2020-12-15
keywords: kubernetes deployment, kubernetes vs docker, K0S
---


We all know and love K3s, right?  It’s now time to discover a new distribution: [k0s](https://k0sproject.io).
<!--more-->


### What’s k0s ?
k0s is a brand new Kubernetes distribution. The current release is 0.8.0. It was published in December 2020.


The latest k0s release:

* Ships a certified and (CIS-benchmarked) Kubernetes 1.19
* Uses containerd as the default container runtime
*  Uses an in-cluster etcd by default and supports SQLite, MySQL (or any compatible), PostgreSQL
* Uses the Calico network plugin by default with network policies
* Enables the Pod Security Policies admission controller
* Uses DNS with CoreDNS
* Exposes cluster metrics via Metrics Server
* Allows the usage of Horizontal Pod Autoscaling (HPA)

A lot of great features will come in future releases, among them:

* Micro VM runtimes (really looking forward to testing this one)
* Zero-downtime cluster upgrades
* Cluster backup and restore
* Air-Gap install
* FIPS 140-2 (coming soon)

We’ll now see how to install k0s. 

### Install singel master
k0s as a single binary acts as the process supervisor for all other control plane components. This means there's no container engine or kubelet running on controllers (by default). Which means there is no way for a cluster user to schedule workloads onto controller nodes.

```
curl -sSLf get.k0s.sh | sudo sh

k0s version

mkdir /etc/k0s
k0s default-config > /etc/k0s/k0s.yaml
```

### Config
In the config file `/etc/k0s/k0s.yaml` you can add helm charts thet will be installed at startup, like prometheus for monitoring or nginx ingress controller.
```
apiVersion: k0s.k0sproject.io/v1beta1
kind: Cluster
metadata:
  name: k0s
spec:
  api:
    address: 192.168.68.106
    sans:
    - my-k0s-control.my-domain.com
  network:
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.96.0.0/12
extensions:
  helm:
    repositories:
    - name: prometheus-community
      url: https://prometheus-community.github.io/helm-charts
    charts:
    - name: prometheus-stack
      chartname: prometheus-community/prometheus
      version: "11.16.8"
      namespace: default
```


```
cat <<EOF > /etc/systemd/system/k0s.service
[Unit]
Description="k0s server"
After=network-online.target
Wants=network-online.target
 
[Service]
Type=simple
ExecStart=/usr/bin/k0s server -c /etc/k0s/k0s.yaml --enable-worker
Restart=always
EOF
```

```
systemctl start k0s.service
systemctl enable k0s.service
journalctl -u k0s.service -f
```


```
sudo curl --output /usr/local/sbin/kubectl -L "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x /usr/local/sbin/kubectl
mkdir ~/.kube
cp /var/lib/k0s/pki/admin.conf ~/.kube/config

kubectl get node
kubectl get po -A
```

```
kubectl run nginx --image=nginx -n default
kubectl get po -A              
```

### Check tge default PSP

```
NAME                PRIV    CAPS   SELINUX    RUNASUSER   FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
00-k0s-privileged   true    *      RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            *
99-k0s-restricted   false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            configMap,downwardAPI,emptyDir,persistentVolumeClaim,projected,secret
```

```
kubectl get psp 99-k0s-restricted -o yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  annotations:
    k0s.k0sproject.io/last-applied-configuration: |
      {"apiVersion":"policy/v1beta1","kind":"PodSecurityPolicy","metadata":{"annotations":null,"name":"99-k0s-restricted"},"spec":{"allowPrivilegeEscalation":false,"allowedCapabilities":[],"fsGroup":{"rule":"RunAsAny"},"hostIPC":false,"hostNetwork":false,"hostPID":false,"privileged":false,"readOnlyRootFilesystem":false,"runAsUser":{"rule":"RunAsAny"},"seLinux":{"rule":"RunAsAny"},"supplementalGroups":{"rule":"RunAsAny"},"volumes":["configMap","downwardAPI","emptyDir","persistentVolumeClaim","projected","secret"]}}
    k0s.k0sproject.io/stack-checksum: b0c62cb2696c6167d7a8289411b06f69
  creationTimestamp: "2020-12-14T17:39:37Z"
  labels:
    k0s.k0sproject.io/stack: defaultpsp
  managedFields:
  - apiVersion: policy/v1beta1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:k0s.k0sproject.io/last-applied-configuration: {}
          f:k0s.k0sproject.io/stack-checksum: {}
        f:labels:
          .: {}
          f:k0s.k0sproject.io/stack: {}
      f:spec:
        f:allowPrivilegeEscalation: {}
        f:fsGroup:
          f:rule: {}
        f:runAsUser:
          f:rule: {}
        f:seLinux:
          f:rule: {}
        f:supplementalGroups:
          f:rule: {}
        f:volumes: {}
    manager: k0s
    operation: Update
    time: "2020-12-14T17:39:37Z"
  name: 99-k0s-restricted
  resourceVersion: "245"
  selfLink: /apis/policy/v1beta1/podsecuritypolicies/99-k0s-restricted
  uid: b59e0bfe-57c2-4b8b-a17b-baa9047a6fcb
spec:
  allowPrivilegeEscalation: false
  fsGroup:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
```

If you check the config file `/etc/k0s/k0s.yaml` you can see it use the 00-k0s-privileged PSP as default and 00-k0s-privileged dose not disable run as root by default. It's sad.

