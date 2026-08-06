---
title: RKE2 The Secure Kubernetes Engine
url: https://devopstales.github.io/kubernetes/rke2-airgap-install/
date: 2020-11-25
keywords: kubernetes deployment, kubernetes vs docker, Kubernetes Secuity, best Practices, rancher
---


In this post I will show you how you can install a secure Kubernetes Engine variant called RKE2 in a Air-Gap environment.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}


### What is RKE2
RKE2, also known as RKE Government, is Rancher's next-generation Kubernetes distribution. It is a fully conformant Kubernetes distribution that focuses on security and compliance within the U.S. Federal Government sector.

### Install RKE2 from rpms
Not like K3S RKE2 offers an rpm repository. Of course in an Air-Gap environment you need an internal repository to sync the packages.

```bash
cat << EOF > /etc/yum.repos.d/rancher-rke2-1-18-latest.repo
[rancher-rke2-common-latest]
name=Rancher RKE2 Common Latest
baseurl=https://rpm.rancher.io/rke2/latest/common/centos/7/noarch
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key

[rancher-rke2-1-18-latest]
name=Rancher RKE2 1.18 Latest
baseurl=https://rpm.rancher.io/rke2/latest/1.18/centos/7/x86_64
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key
EOF

yum -y install rke2-server nano
```


In an Air-Gap environment you cannot connect to the public internet so the containerd engine cannot connest to the registry. In this scenario yo have two options. Create an internal registry and upload all images or import images from tarball. In this demo I will use the second option.

```bash
mkdir -p /var/lib/rancher/rke2/agent/images/
scp rke2-images.linux-amd64.tar masern01:/var/lib/rancher/rke2/agent/images/
cd /var/lib/rancher/rke2/agent/images/
```

For RKE2 you didn't nee docker engine. The rpms will install all the necessary binaris to run a container.  
```bash
echo 'PATH=$PATH:/usr/local/bin' >> /etc/profile
echo 'PATH=$PATH:/var/lib/rancher/rke2/bin' >> /etc/profile
source /etc/profile
```

```bash
setenforce 1
getenforce
sed -i 's/=\(disabled\|permissive\)/=enforcing/g' /etc/sysconfig/selinux
systemctl start firewalld
systemctl enable firewalld
```

For the demo I will use `firewalld` to block all outgoing request from the server. This is how I emulate the Air-Gap environment.
```bash
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 0 -p tcp -m tcp --dport=443 -j DROP
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 0 -p tcp -m tcp --dport=80 -j DROP
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 1 -j ACCEPT
firewall-cmd --reload
```


Enable hardened mode.
```bash
mkdir -p /etc/rancher/rke2
cat << EOF >  /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
profile: "cis-1.5"
selinux: true
EOF


sudo cp -f /usr/share/rke2/rke2-cis-sysctl.conf /etc/sysctl.d/60-rke2-cis.conf
sysctl -p /etc/sysctl.d/60-rke2-cis.conf

useradd -r -c "etcd user" -s /sbin/nologin -M etcd
```

On my VM there is multiple network interface So I will configure what to use the kubernetes engine.
```bash
mkdir -p /var/lib/rancher/rke2/server/manifests/
cat << EOF > /var/lib/rancher/rke2/server/manifests/rke2-canal-config.yml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-canal
  namespace: kube-system
spec:
  valuesContent: |-
    flannel:
      iface: "enp0s8"
EOF

cat << EOF > /var/lib/rancher/rke2/server/manifests/rke2-ingress-nginx-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-ingress-nginx
  namespace: kube-system
spec:
  valuesContent: |-
    controller:
      metrics:
        enable: true
        service:
          annotations:
            prometheus.io/scrape: "true"
            prometheus.io/port: "10254"
EOF

cat << EOF > /var/lib/rancher/rke2/server/manifests/rke2-kube-proxy-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-kube-proxy
  namespace: kube-system
spec:
  valuesContent: |-
    metricsBindAddress: 0.0.0.0:10249
EOF
```

```bash
systemctl enable rke2-server.service
systemctl start rke2-server.service
journalctl -u rke2-server -f


mkdir ~/.kube
ln -s /etc/rancher/rke2/rke2.yaml ~/.kube/config
ln -s /var/lib/rancher/rke2/agent/etc/crictl.yaml /etc/crictl.yaml
chmod 600 ~/.kube/config

kubectl get node
crictl ps
crictl images

kubectl edit psp global-restricted-psp
# remove apparmor lines in annotation and save

### Autodeploy folder
/var/lib/rancher/rke2/server/manifests/
```

