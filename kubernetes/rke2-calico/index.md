---
title: RKE2 Install With Calico
url: https://devopstales.github.io/kubernetes/rke2-calico/
date: 2021-05-25
keywords: kubernetes deployment, kubernetes vs docker, Kubernetes Secuity, best Practices, rancher, Calico, ebpf
---


In this post I will show you how you can install a RKE2 with Calico and encripted VXLAN.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

## RKE2 Setup

### Project Longhorn Prerequisites
```bash
yum install -y epel-release
yum install -y nano curl wget git tmux jq
yum install -y iscsi-initiator-utils 
modprobe iscsi_tcp
echo "iscsi_tcp" >/etc/modules-load.d/iscsi-tcp.conf
systemctl enable iscsid
systemctl start iscsid 
```

Ensure the eBFP filesystem is mounted (which should already be the case on RHEL 8.3):
```bash
mount | grep /sys/fs/bpf
# if present should output, e.g. "none on /sys/fs/bpf type bpf"...
```

If that's not the case, mount it using the commands down here:
```bash
sudo mount bpffs -t bpf /sys/fs/bpf
sudo bash -c 'cat <<EOF >> /etc/fstab
none /sys/fs/bpf bpf rw,relatime 0 0
EOF'
```

```bash
cat <<EOF>> /etc/NetworkManager/conf.d/rke2-canal.conf
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:flannel*
EOF
systemctl reload NetworkManager
```

### RKE2 rpm Install
```bash
cat << EOF > /etc/yum.repos.d/rancher-rke2-1-20-latest.repo
[rancher-rke2-common-latest]
name=Rancher RKE2 Common Latest
baseurl=https://rpm.rancher.io/rke2/latest/common/centos/8/noarch
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key

[rancher-rke2-1-20-latest]
name=Rancher RKE2 1.20 Latest
baseurl=https://rpm.rancher.io/rke2/latest/1.20/centos/8/x86_64
enabled=1
gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key
EOF

yum -y install rke2-server
```

### Kubectl, Helm & RKE2
Install `kubectl`, `helm` and RKE2 to the host system:
```bash
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
echo 'PATH=$PATH:/usr/local/bin' >> /etc/profile
echo 'PATH=$PATH:/var/lib/rancher/rke2/bin' >> /etc/profile
source /etc/profile

sudo dnf copr -y enable cerenit/helm
sudo dnf install -y helm
```

### RKE2 specific ports
```bash
sudo firewall-cmd --add-port=9345/tcp --permanent
sudo firewall-cmd --add-port=6443/tcp --permanent
sudo firewall-cmd --add-port=10250/tcp --permanent
sudo firewall-cmd --add-port=2379/tcp --permanent
sudo firewall-cmd --add-port=2380/tcp --permanent
sudo firewall-cmd --add-port=30000-32767/tcp --permanent
# Used for the Rancher Monitoring
sudo firewall-cmd --add-port=9796/tcp --permanent
sudo firewall-cmd --add-port=19090/tcp --permanent
sudo firewall-cmd --add-port=6942/tcp --permanent
sudo firewall-cmd --add-port=9091/tcp --permanent
### CNI specific ports
sudo firewall-cmd --add-port=179/tcp --permanent
sudo firewall-cmd --add-port=4789/udp --permanent
sudo firewall-cmd --add-port=5473/tcp --permanent
sudo firewall-cmd --add-port=9098/tcp --permanent
sudo firewall-cmd --add-port=9099/tcp --permanent
sudo firewall-cmd --add-port=5473/tcp --permanent
### Ingress Controller specific ports
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --add-port=443/tcp --permanent
### To get DNS resolution working, simply enable Masquerading.
sudo firewall-cmd --zone=public  --add-masquerade --permanent

### Finally apply all the firewall changes
sudo firewall-cmd --reload
```

Verification:
```bash
sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eno1
  sources: 
  services: cockpit dhcpv6-client ssh wireguard
  ports: 9345/tcp 6443/tcp 10250/tcp 2379/tcp 2380/tcp 30000-32767/tcp 4240/tcp 6081/udp 80/tcp 443/tcp 4244/tcp 9796/tcp 19090/tcp 6942/tcp 9091/tcp
  protocols: 
  masquerade: yes
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

### Basic Configuration
```bash
mkdir -p /etc/rancher/rke2
cat << EOF >  /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
profile: "cis-1.5"
selinux: true
# add ips/hostname of hosts and loadbalancer
tls-san:
  - "k8s.mydomain.intra"
  - "172.17.9.10"
# Make a etcd snapshot every 6 hours
etcd-snapshot-schedule-cron: "0 */6 * * *"
# Keep 56 etcd snapshorts (equals to 2 weeks with 6 a day)
etcd-snapshot-retention: 56
cni:
  - calico
disable:
  - rke2-canal
  - rke2-kube-proxy
EOF
```

**Note:** I disabled `rke2-canal` and `rke2-kube-proxy` since I plan to install Canal as CNI in ["kube-proxy less mode"](https://docs.projectcalico.org/maintenance/ebpf/enabling-bpf). Do not disable `rke2-kube-proxy` if you use another CNI - it will not work afterwards!

```bash
sudo cp -f /usr/share/rke2/rke2-cis-sysctl.conf /etc/sysctl.d/60-rke2-cis.conf
sysctl -p /etc/sysctl.d/60-rke2-cis.conf

useradd -r -c "etcd user" -s /sbin/nologin -M etcd

mkdir -p /var/lib/rancher/rke2/server/manifests/
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
        service:
          annotations:
            prometheus.io/scrape: "true"
            prometheus.io/port: "10254"
EOF
```

```bash
kubectl get endpoints kubernetes -o wide
NAME         ENDPOINTS        AGE
kubernetes   10.0.2.15:6443   86m
```

### Prevent RKE2 Package Updates
In order to provide more stability, I chose to DNF/YUM "mark/hold" the RKE2 related packages so a `dnf update`/`yum update` does not mess around with them.

Add the following line to `/etc/dnf/dnf.conf` and/or `/etc/yum.conf`:
```bash
exclude=rke2-*
```

## Starting RKE2
Enable the `rke2-server` service and start it:
```bash
sudo systemctl enable rke2-server --now
```

Verification:
```bash
sudo systemctl status rke2-server
sudo journalctl -u rke2-server -f
```

## Configure Kubectl (on RKE2 Host)
```bash
mkdir ~/.kube
ln -s /etc/rancher/rke2/rke2.yaml ~/.kube/config
chmod 600 /root/.kube/config
ln -s /var/lib/rancher/rke2/agent/etc/crictl.yaml /etc/crictl.yaml

kubectl get node
crictl ps
crictl images
```

Verification:
```bash
kubectl get nodes
NAME                    STATUS   ROLES         AGE     VERSION
k8s.mydomain.intra   Ready   etcd,master   2m4s   v1.18.16+rke2r1
```


