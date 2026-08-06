---
title: Install an OpenShift 4 cluster with Calico
url: https://devopstales.github.io/kubernetes/openshift4-calico/
date: 2022-03-12
keywords: OpenShift, OpenShift 4, calico, OKD, OKD4
---


In this Post I will show you How you can Install  OpenShift 4 cluster with Calico.
<!--more-->

{{< content "/filedir/openshift4.html" >}}

### Infrastructure

| Host | ROLES | OS | IP |
|:------:|:------:|:------:|:------:|
| pfsense | Load Balancer, dhcp, dns | pfsense | 192.168.1.1 |
| okd4-services | pxeboot | CentOS 7 | 192.168.1.200 |
| okd4-bootstrap | bootstrap | Fedora Core OS | 192.168.1.210 |
| okd4-mastr-1 | master | Fedora Core OS | 192.168.1.201 |
| okd4-mastr-2 | master | Fedora Core OS | 192.168.1.202 |
| okd4-mastr-3 | master | Fedora Core OS | 192.168.1.203 |
| okd4-worker-1 | worker | Fedora Core OS | 192.168.1.204 |
| okd4-worker-2 | worker | Fedora Core OS | 192.168.1.205 |
| okd4-worker-4 | worker | Fedora Core OS | 192.168.1.206 |
| okd4-worker-5 | worker | Fedora Core OS | 192.168.1.207 |

### DNS Config

```bash
; OpenShift Container Platform Cluster - A records
pfsense.okd.mydomain.intra.          IN      A      192.168.1.1
okd4-bootstrap.okd.mydomain.intra.   IN      A      192.168.1.210

okd4-mastr-1.okd.mydomain.intra.        IN      A      192.168.1.201
okd4-mastr-2.okd.mydomain.intra.        IN      A      192.168.1.202
okd4-mastr-3.okd.mydomain.intra.        IN      A      192.168.1.203
okd4-worker-1.okd.mydomain.intra.        IN      A      192.168.1.204
okd4-worker-2.okd.mydomain.intra.        IN      A      192.168.1.205
okd4-worker-3.okd.mydomain.intra.        IN      A      192.168.1.206
okd4-worker-4.okd.mydomain.intra.        IN      A      192.168.1.207


; OpenShift internal cluster IPs - A records
api.okd.mydomain.intra.            IN      A      192.168.1.1
api-int.okd.mydomain.intra.        IN      A      192.168.1.1
etcd-0.okd.mydomain.intra.         IN      A     192.168.1.201
etcd-1.okd.mydomain.intra.         IN      A     192.168.1.202
etcd-2.okd.mydomain.intra.         IN      A     192.168.1.203

okd.mydomain.intra.                IN      A      192.168.1.1
*.okd.mydomain.intra.              IN      A      192.168.1.1

; OpenShift internal cluster IPs - SRV records
_etcd-server-ssl._tcp.okd.mydomain.intra.    86400     IN    SRV     0    10    2380    etcd-0.okd.mydomain.intra.
_etcd-server-ssl._tcp.okd.mydomain.intra.    86400     IN    SRV     0    10    2380    etcd-1.okd.mydomain.intra.
_etcd-server-ssl._tcp.okd.mydomain.intra.    86400     IN    SRV     0    10    2380    etcd-2.okd.mydomain.intra.
```

### DHCP Config:
```bash
32:89:07:57:27:00  192.168.1.200 	okd4-services
32:89:07:57:27:10  192.168.1.210 	okd4-bootstrap
32:89:07:57:27:01  192.168.1.201 	okd4-mastr-1
32:89:07:57:27:02  192.168.1.202 	okd4-mastr-2
32:89:07:57:27:03  192.168.1.203 	okd4-mastr-3
32:89:07:57:27:04  192.168.1.204 	okd4-worker-1
32:89:07:57:27:05  192.168.1.205 	okd4-worker-2
32:89:07:57:27:06  192.168.1.206 	okd4-worker-3
32:89:07:57:27:07  192.168.1.207 	okd4-worker-4
```

```bash
Next Server: 192.168.1.200
Default BIOS file name: pxelinux.0
```

## HAPROXY Config:
```bash
192.168.201.1 6443  -->  192.168.1.210   6443
192.168.201.1 6443  -->  192.168.1.201  6443
192.168.201.1 6443  -->  192.168.1.202  6443
192.168.201.1 6443  -->  192.168.1.202  6443
192.168.201.1 22623 -->  192.168.1.210   22623
192.168.201.1 22623 -->  192.168.1.201  22623
192.168.201.1 22623 -->  192.168.1.202  22623
192.168.201.1 22623 -->  192.168.1.202  22623
192.168.201.1 80    -->  192.168.1.204  80
192.168.201.1 80    -->  192.168.1.205  80
192.168.201.1 443   -->  192.168.1.204  443
192.168.201.1 443   -->  192.168.1.205  443
<publicip> 80    -->  192.168.1.206  80
<publicip> 80    -->  192.168.1.207  80
<publicip> 443   -->  192.168.1.206  443
<publicip> 443   -->  192.168.1.207  443
```

### Install and configure pxeboot

```bash
ssh okd4-services

yum install epel-release -y
yum install httpd nano jq -y
dnf install -y tftp-server syslinux-tftpboot
```

```bash
mkdir -p /var/lib/tftpboot
cp -v /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/
cp -v /usr/share/syslinux/menu.c32 /var/lib/tftpboot/
cp -v /usr/share/syslinux/mboot.c32 /var/lib/tftpboot/
cp -v /usr/share/syslinux/chain.c32 /var/lib/tftpboot/
cp -v /usr/share/syslinux/ldlinux.c32 /var/lib/tftpboot/
cp -v /usr/share/syslinux/libutil.c32 /var/lib/tftpboot/

mkdir -p /var/lib/tftpboot/fcsos33
cd /var/lib/tftpboot/fcsos33
RHCOS_BASEURL=https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/
wget ${RHCOS_BASEURL}33.20210117.3.2/x86_64/fedora-coreos-33.20210117.3.2-live-kernel-x86_64
wget ${RHCOS_BASEURL}33.20210117.3.2/x86_64/fedora-coreos-33.20210117.3.2-live-initramfs.x86_64.img
wget ${RHCOS_BASEURL}33.20210117.3.2/x86_64/fedora-coreos-33.20210117.3.2-live-rootfs.x86_64.img
cd ~

mkdir /var/lib/tftpboot/pxelinux.cfg
```

```bash
cat > /var/lib/tftpboot/pxelinux.cfg/default << EOF
default menu.c32
prompt 0
timeout 30
menu title PXE Menu

label 1
menu label ^1) Boot from local drive
localboot 0x00

label 2
menu label ^2) Install OKD Bootstrap
KERNEL /fcsos33/fedora-coreos-33.20210117.3.2-live-kernel-x86_64
APPEND initrd=/fcsos33/fedora-coreos-33.20210117.3.2-live-initramfs.x86_64.img,/fcsos33/fedora-coreos-33.20210117.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/vda coreos.inst.image_url=http://192.168.201.4/fcos.raw.xz coreos.inst.ignition_url=http://192.168.201.4/bootstrap.ign

label 3
menu label ^3) Install OKD Master
KERNEL /fcsos33/fedora-coreos-33.20210117.3.2-live-kernel-x86_64
APPEND initrd=/fcsos33/fedora-coreos-33.20210117.3.2-live-initramfs.x86_64.img,/fcsos33/fedora-coreos-33.20210117.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/vda coreos.inst.image_url=http://192.168.201.4/fcos.raw.xz coreos.inst.ignition_url=http://192.168.201.4/master.ign

label 4
menu label ^4) Install OKD Worker
KERNEL /fcsos33/fedora-coreos-33.20210117.3.2-live-kernel-x86_64
APPEND initrd=/fcsos33/fedora-coreos-33.20210117.3.2-live-initramfs.x86_64.img,/fcsos33/fedora-coreos-33.20210117.3.2-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/vda coreos.inst.image_url=http://192.168.201.4/fcos.raw.xz coreos.inst.ignition_url=http://192.168.201.4/worker.ign
EOF
```

Then run: `systemctl enable --now tftp.service`

### Create okd config

> find the raw image:
> https://getfedora.org/en/coreos/download?tab=metal_virtualized&stream=stable
> 4K vs non 4K

```bash
wget https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/33.20210117.3.2/x86_64/fedora-coreos-33.20210117.3.2-metal.x86_64.raw.xz
wget https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/33.20210117.3.2/x86_64/fedora-coreos-33.20210117.3.2-metal.x86_64.raw.xz.sig
cp fedora-coreos-33.20210117.3.2-metal.x86_64.raw.xz /var/www/html/fcos.raw.xz
cp fedora-coreos-33.20210117.3.2-metal.x86_64.raw.xz.sig /var/www/html/fcos.raw.xz.sig
```

```bash
# find installer
# https://github.com/openshift/okd/releases

wget https://github.com/openshift/okd/releases/download/4.6.0-0.okd-2021-02-14-205305/openshift-client-linux-4.6.0-0.okd-2021-02-14-205305.tar.gz
wget https://github.com/openshift/okd/releases/download/4.6.0-0.okd-2021-02-14-205305/openshift-install-linux-4.6.0-0.okd-2021-02-14-205305.tar.gz

tar -xzf openshift-client-linux-4.6.0-0.okd-2021-02-14-205305.tar.gz
tar -xzf openshift-install-linux-4.6.0-0.okd-2021-02-14-205305.tar.gz

sudo mv kubectl oc openshift-install /usr/local/bin/
oc version
openshift-install version

mkdir install_dir
```

Use Calico for `networkType`.

```bash
cat > install_dir/install-config.yaml << EOF
apiVersion: v1
baseDomain: mydomain.intra
metadata:
  name: okd

compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0

controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3

networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: Calico
  serviceNetwork:
  - 172.30.0.0/16

platform:
  none: {}

fips: false

pullSecret: '{"auths":{"fake":{"auth": "bar"}}}'
sshKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDK7lDozs9WLJD14H+nz...' 
EOF
```

```bash
openshift-install create manifests --dir=install_dir/
```

You may want to provide Calico with additional configuration at install-time. For example, BGP configuration or peers. You can use a Kubernetes ConfigMap with your desired Calico resources in order to set configuration as part of the installation.

To include [Calico resources](https://projectcalico.docs.tigera.io/reference/resources/) during installation, edit `install_dir/manifests/02-configmap-calico-resources.yaml` in order to add your own configuration.

```bash
oc create configmap -n tigera-operator calico-resources \
  --from-file=<resource-directory> --dry-run -o yaml \
  > install_dir/manifests/02-configmap-calico-resources.yaml
```

```bash
sed -i 's/mastersSchedulable: true/mastersSchedulable: False/' install_dir/manifests/cluster-scheduler-02-config.yml

openshift-install create ignition-configs --dir=install_dir/

sudo cp -R install_dir/*.ign /var/www/html/
sudo cp -R install_dir/metadata.json /var/www/html/
sudo chown -R apache: /var/www/html/
sudo chmod -R 755 /var/www/html/
```

> The config contains certificates that is walid for 24 hours.

### Starting the VMs
It's time to start the VMs. Select the okd4-bootstrap VM and navigate to Console. Start the VM. Then one by one the masters and the workers too.

### Bootstrap OKD Cluster

You can monitor the installation progress by running the following command.
```bash
openshift-install --dir=install_dir/ wait-for bootstrap-complete --log-level=info
```

> The certificates in the cluster is not authomaticle approved so I use the abow `tmux` command to approve

```bash
tmux
export KUBECONFIG=~/install_dir/auth/kubeconfig
while true; do echo `oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs -r oc adm certificate approve`; sleep 60; done
```


Once the bootstrap process completes, you should see the following messages.

```bash
INFO It is now safe to remove the bootstrap resources
```

Then stop the bootstrap node.

```bash
# debug command to check the health of the cluster.
watch oc get csr
watch oc get node

oc get clusteroperator
oc get clusterversion

watch "oc get clusteroperator"
watch "oc get po -A | grep -v Running | grep -v Completed"


curl -X GET https://api.okd.mydomain.intra:6443/healthz -k
```

Verify Calico is installed by verifying the components are available with the following command.

```bash
oc get tigerastatus
```

Wait for the console to be available. Once it is available, we can point a browser to https://console-openshift-console.okd.mydomain.intra

You will get an SSL error because the certificate is not valid for this domain. That's normal. Just bypass the SSL error.

Login with user "kubeadmin".You can find the kubeadmin password in a file generated during the installation.

```bash
cat install_dir/auth/kubeadmin-password
```

Optionally you can integrate with Operator Lifecycle Manager (OLM). First you will need to create an OperatorGroup for the operator:

```yaml
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: tigera-operator
  namespace: tigera-operator
spec:
  targetNamespaces:
    - tigera-operator
EOF
```

Next, you will create a Subscription to the operator.
```yaml
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: tigera-operator
  namespace: tigera-operator
spec:
  channel: release-v1.25
  installPlanApproval: Manual
  name: tigera-operator
  source: certified-operators
  sourceNamespace: openshift-marketplace
  startingCSV: tigera-operator.v1.25.3
EOF
```

Finally, log in to the OpenShift console, navigate to the Installed Operators section and approve the Install Plan for the operator.

