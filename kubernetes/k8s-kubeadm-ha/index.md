---
title: How to install kubernetes with kubeadm in HA mode
url: https://devopstales.github.io/kubernetes/k8s-kubeadm-ha/
date: 2020-04-02
keywords: kubernetes deployment, kubernetes vs docker, kubernetes HA
---


In this post I will show you how to install kubernetes in HA mode with kubeadm, keepaliwed and envoyproxy.
<!--more-->

{{< content "/filedir/kubernetes.html" >}}

```
172.17.8.100  # kubernetes cluster ip
172.17.8.101  master01 # master node
172.17.8.102  master02 # frontend node
172.17.8.103  master03 # worker node

# hardware requirement
2 CPU
4G RAM
```

### Install Docker
```
yum install -y -q yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y -q docker-ce docker-compose

mkdir /etc/docker
echo '{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}' > /etc/docker/daemon.json

systemctl enable docker
systemctl start docker
```

### Disable swap
```
free -h
swapoff -a
swapoff -a
sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab
free -h
```

### Configuuration
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.ipv6.conf.all.disable_ipv6      = 1
net.ipv6.conf.default.disable_ipv6  = 1
EOF
cat >>/etc/sysctl.d/ipv6.conf<<EOF
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv6.conf.eth0.disable_ipv6 = 1
EOF
sysctl --system
```

### Install kubeadm
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF


yum install epel-release -y
yum install -y kubeadm kubelet kubectl keepalived
```

### Configure keepalived on first master

```
touch /etc/keepalived/check_apiserver.sh
chmod +x /etc/keepalived/check_apiserver.sh

cat <<EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id k8s-node1
    enable_script_security
    script_user root
}
vrrp_script check_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -10
    fall 2
    rise 2
}
vrrp_instance VI_1 {
    state BACKUP
    interface enp0s8
    mcast_src_ip 172.17.8.101
    virtual_router_id 51
    priority 150
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass Password1
    }
    virtual_ipaddress {
        172.17.8.100/24 brd 172.17.8.255 dev enp0s8
    }
    track_script {
       check_apiserver
    }
}
EOF

systemctl start keepalived
systemctl enable keepalived
```

### Configure envoy on first master
```
cat <<EOF > /etc/kubernetes/envoy.yaml
static_resources:
  listeners:
  - name: main
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 16443
    filter_chains:
    - filters:
      - name: envoy.tcp_proxy
        config:
          stat_prefix: ingress_tcp
          cluster: k8s

  clusters:
  - name: k8s
    connect_timeout: 0.25s
    type: strict_dns
    lb_policy: round_robin
    hosts:
    - socket_address:
        address: 172.17.8.101
        port_value: 6443
    - socket_address:
        address: 172.17.8.102
        port_value: 6443
    - socket_address:
        address: 172.17.8.103
        port_value: 6443
    health_checks:
    - timeout: 1s
      interval: 5s
      unhealthy_threshold: 1
      healthy_threshold: 1
      http_health_check:
        path: "/healthz"

admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 8001
EOF
```

### Create loadbalancer on all masters
```

cat<<EOF > /etc/kubernetes/envoy.yaml
static_resources:
  listeners:
  - name: main
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 16443
    filter_chains:
    - filters:
      - name: envoy.tcp_proxy
        config:
          stat_prefix: ingress_tcp
          cluster: k8s

  clusters:
  - name: k8s
    connect_timeout: 0.25s
    type: strict_dns # static
    lb_policy: round_robin
    hosts:
    - socket_address:
        address: 172.17.8.101
        port_value: 6443
    - socket_address:
        address: 172.17.8.102
        port_value: 6443
    - socket_address:
        address: 172.17.8.103
        port_value: 6443
    health_checks:
    - timeout: 1s
      interval: 5s
      unhealthy_threshold: 1
      healthy_threshold: 1
      http_health_check:
        path: "/healthz"

admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 8001
EOF

cat<<EOF > /etc/kubernetes/docker-compose.yaml
version: '3'
services:
    api-lb:
        image: envoyproxy/envoy:latest
        restart: always
        network_mode: "host"
        ports:
            - 16443:16443
            - 8001:8001
        volumes:
            - /etc/kubernetes/envoy.yaml:/etc/envoy/envoy.yaml
EOF

cd /etc/kubernetes/
docker-compose pull
docker-compose up -d
docker-compose ps

netstat -tulpn | grep 6443
```

### Initialize kubernetes in the first master

I have multiple interfaces in my masters so to use the correct one I need to add `--apiserver-advertise-address "172.17.8.101"` to my kubeadm commands and add `KUBELET_EXTRA_ARGS` for kubelet config.
```
echo 'KUBELET_EXTRA_ARGS="--node-ip=172.17.8.101"' > /etc/sysconfig/kubelet

kubeadm config images pull --kubernetes-version 1.16.8
kubeadm init --control-plane-endpoint "172.17.8.100:16443" --apiserver-advertise-address "172.17.8.101" --upload-certs --kubernetes-version 1.16.8 --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get no

kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```

### Join other masters

```

kubeadm config images pull --kubernetes-version 1.16.8

echo 'KUBELET_EXTRA_ARGS="--node-ip=172.17.8.102"' > /etc/sysconfig/kubelet

kubeadm join 172.17.8.100:16443 --token 3vqtop.z2kbok4o0wchu4ed \
  --discovery-token-ca-cert-hash sha256:5840ee4de07bb296e2639669c17df7e3240271a1880115336ebc5b91fb8a3555 \
  --control-plane --certificate-key dc99dc10a0269d1a3edfc2e318a78c6bbebdee8081b460535f699d210cec5dcb \
  --apiserver-advertise-address "172.17.8.102"

echo 'KUBELET_EXTRA_ARGS="--node-ip=172.17.8.103"' > /etc/sysconfig/kubelet

kubeadm join 172.17.8.100:16443 --token 3vqtop.z2kbok4o0wchu4ed \
  --discovery-token-ca-cert-hash sha256:5840ee4de07bb296e2639669c17df7e3240271a1880115336ebc5b91fb8a3555 \
  --control-plane --certificate-key dc99dc10a0269d1a3edfc2e318a78c6bbebdee8081b460535f699d210cec5dcb \
  --apiserver-advertise-address "172.17.8.103"
```

### Fix keepalibed check script on first master
```
echo '#!/bin/bash

# if check error then repeat check for 12 times, else exit
err=0
for k in $(seq 1 12)
do
    check_code=$(curl -sk https://localhost:16443)
    if [[ $check_code == "" ]]; then
        err=$(expr $err + 1)
        sleep 5
        continue
    else
        err=0
        break
    fi
done

if [[ $err != "0" ]]; then
    # if apiserver is down send SIG=1
    echo 'apiserver error!'
    exit 1
else
    # if apiserver is up send SIG=0
    echo 'apiserver normal!'
    exit 0
fi' > /etc/keepalived/check_apiserver.sh
```

### Configure keepalived on other masters

```
echo '#!/bin/bash

# if check error then repeat check for 12 times, else exit
err=0
for k in $(seq 1 12)
do
    check_code=$(curl -sk https://localhost:16443)
    if [[ $check_code == "" ]]; then
        err=$(expr $err + 1)
        sleep 5
        continue
    else
        err=0
        break
    fi
done

if [[ $err != "0" ]]; then
    # if apiserver is down send SIG=1
    echo 'apiserver error!'
    exit 1
else
    # if apiserver is up send SIG=0
    echo 'apiserver normal!'
    exit 0
fi' > /etc/keepalived/check_apiserver.sh

chmod +x /etc/keepalived/check_apiserver.sh
```

```
cat <<EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id k8s-node2
    enable_script_security
    script_user root
}
vrrp_script check_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -10
    fall 2
    rise 2
}
vrrp_instance VI_1 {
    state BACKUP
    interface enp0s8
    mcast_src_ip 172.17.8.102
    virtual_router_id 51
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass Password1
    }
    virtual_ipaddress {
        172.17.8.100/24 brd 172.17.8.255 dev enp0s8
    }
    track_script {
       check_apiserver
    }
}
EOF

systemctl start keepalived
systemctl enable keepalived
```

```
cat <<EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id k8s-node3
    enable_script_security
    script_user root
}
vrrp_script check_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -10
    fall 2
    rise 2
}
vrrp_instance VI_1 {
    state BACKUP
    interface enp0s8
    mcast_src_ip 172.17.8.103
    virtual_router_id 51
    priority 50
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass Password1
    }
    virtual_ipaddress {
        172.17.8.100/24 brd 172.17.8.255 dev enp0s8
    }
    track_script {
       check_apiserver
    }
}
EOF

systemctl start keepalived
systemctl enable keepalived
```

```
kubectl scale deploy/coredns  --replicas=3 -n kube-system
```

