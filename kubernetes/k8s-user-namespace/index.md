---
title: Linux user namespace management wit CRI-O in Kubernetes
url: https://devopstales.github.io/kubernetes/k8s-user-namespace/
date: 2022-11-02
keywords: CRI-O, kubernetes deployment, user namespace
---


In this blog post I will introduce user namespaces, then I will show you how you can use it in Kubernetes.

<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

# What are user namespaces?

As we talked about in a [prewious post]() container engines uses the linux kernels namespaces to isolate the conatiners. For example, two containers in different network namespaces will not see each other’s network interfaces. Two containers in different PID namespaces will not see each other’s processes.

On Linux, all files and all processes are owned by a specific user id and group id, usually defined in `/etc/passwd` and `/etc/group`. [User namespaces](https://man7.org/linux/man-pages/man7/user_namespaces.7.html)  are isolates user IDs and group IDs from each other. With user namespaces the container engine can let a container only see a subset of the host’s user IDs and group IDs.

## Why this is important

By default the container engines share the same user namespace in the container as the host use. So If I use the root user with the 0 user ID in a container it is the same ID as he root user use on the host. So if an unprivileged user on a hos has the ability to run containers with this security loophole it can make changes on the host without sudo privilege on the host:

```bash
$ docker run -v /etc/:/etc/ -ti ubuntu
root@6803a66e58d0:/# passwd
New password:
Retype new password:
passwd: password updated successfully
root@6803a66e58d0:/# exit

$ su -
Password:
Hello, DevOpsTales! You are a sysadmin now
#
```

## The solution rootless mode

The firs container engine that can be used in rootless mode was **podman** they used the `subuid` and `bunguid` to run containers in rootless mode. Normally a user or group has only one ID, but wit `subuid` and `bunguid` you can allocate an ID segment for the user or groupe.

```bash
====================================================================
User Specification
====================================================================

# The following below commands allocates the UIDs and GIDs from 100000to 165535 to the podman user and group respectively.

$ sudo touch /etc/{subgid,subuid}
$ sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 ${USER}
$ grep ${USER} /etc/subuid /etc/subgid
/etc/subuid:${USER}:100000:65536
/etc/subgid:${USER}:100000:65536
```
Now bot **podman** and **docker** can be installed in rootless mode. The only problem with rootless mode that Kubernets can not use it.

## Kubernetes user namespace management with CRI-O

To solve this problem **CRI-O** [added support](https://github.com/cri-o/cri-o/pull/3944) for user namespace configuration through pod annotations.

```bash
VERSION=1.25

sudo curl -L -o /etc/yum.repos.d/devel_kubic_libcontainers_stable.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/CentOS_8/devel:kubic:libcontainers:stable.repo
sudo curl -L -o /etc/yum.repos.d/devel_kubic_libcontainers_stable_cri-o_${VERSION}.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/${VERSION}/CentOS_8/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo

yum install cri-o nano wget
```

```bas
cat <<'EOF' | sudo tee /etc/modules-load.d/crio.conf > /dev/null
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
```

```bash
free -h
swapoff -a
swapoff -a
sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab
free -h
```

```bash
# Configure User anmespacing in CRI-O
mkdir /etc/crio/crio.conf.d/
cat <<EOF > /etc/crio/crio.conf.d/01-userns-workload.conf
[crio.runtime.workloads.userns]
activation_annotation = "io.kubernetes.cri-o.userns-mode"
allowed_annotations = ["io.kubernetes.cri-o.userns-mode"]
EOF

nano /etc/containers/registries.conf
unqualified-search-registries = ["registry.access.redhat.com", "registry.redhat.io", "quay.io", "docker.io"]

sed -i.bak -r 's/network_backend = "cni"/#network_backend = ""/' /usr/share/containers/containers.conf
```

Change the pod-subnet `10.244.0.0/16` in the cri-o bridge config `/etc/cni/net.d/100-crio-bridge.conf`

```json
nano /etc/cni/net.d/100-crio-bridge.conf
{
    "cniVersion": "0.3.1",
    "name": "crio",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "hairpinMode": true,
    "ipam": {
	"type": "host-local",
        "routes": [
            { "dst": "0.0.0.0/0" },
            { "dst": "1100:200::1/24" }
        ],
	"ranges": [
            [{ "subnet": "10.244.0.0/16" }]
        ]
    }
}
```

The **CRI-O** will run the containers with the `containers` user so I need to create `/etc/subuid` and `/etc/subgid` on nodes.

> SubUID/GIDs are a range of user/group IDs that a user is allowed to use.

```bash
echo "containers:200000:268435456" >> /etc/subuid
echo "containers:200000:268435456" >> /etc/subgid
```

> First I created the id ranges for `root` user because **CRI-O** runs as `root` Fu the I find the fallowing ERROR in the **CRI-O** log:
> ```
> Cannot find mappings for user \"containers\": No subuid
> ranges found for user \"containers\" in /etc/subuid"
> ```

### Install Kubernetes

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

```bash
CRIP_VERSION=$(crio --version | egrep ^Version | awk '{print $2}')
yum install kubelet-$CRIP_VERSION kubeadm-$CRIP_VERSION kubectl-$CRIP_VERSION cri-tools iproute-tc -y
```

```bash
IP=192.168.200.10

mkdir /var/lib/kubelet/

# --node-ip for multi interface configuration
cat <<EOF > /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--node-ip='$IP'"
EOF
```


```bash
systemctl enable kubelet.service
systemctl enable --now crio
kubeadm config images pull --cri-socket=unix:///var/run/crio/crio.sock --kubernetes-version=$CRIP_VERSION
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=$IP  --kubernetes-version=$CRIP_VERSION --cri-socket=unix:///var/run/crio/crio.sock

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


crictl info
kubectl get node -o wide
kubectl get po --all-namespaces

kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

 and **docker/containerd** can be install in rootless mode.  user namespaces it.

### Demo time

```bash
nano pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: not-userns-pod
  annotations:
    io.kubernetes.cri-o.userns-mode: "auto" # this will not work
spec:
  containers:
  - command:
    - sleep
    - 2d
    image: registry.fedoraproject.org/fedora-minimal
    name: not-userns-ctr
    imagePullPolicy: IfNotPresent
status: {}
---
apiVersion: v1
kind: Pod
metadata:
  name: standard-pod
spec:
  containers:
  - command:
    - sleep
    - 3d
    image: registry.fedoraproject.org/fedora-minimal
    name: not-userns-ctr
    imagePullPolicy: IfNotPresent
status: {}
```

```bash
kubectl aply -f pod.yaml

ps -eo args,pid | grep sleep
sleep 2d                      57277
sleep 3d                      58878
grep --color=auto sleep       58918
```

```bash
# standard container
cat /proc/58878/uid_map
         0          0 4294967295

# namespaced container
cat /proc/57277/uid_map
         0     200000      65536
```

As you can see the container's uid range is shifted.

