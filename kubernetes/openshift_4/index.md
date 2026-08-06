---
title: OpenShift 4.2 with Red Hat CodeReady Containers
url: https://devopstales.github.io/kubernetes/openshift_4/
date: 2020-03-02
keywords: ansible, openshift 4.2, openshift tutorial, openshift container platform, rad hat openshift
---


The Red Hat CodeReady Containers enables you to run a minimal OpenShift 4.2 or newer cluster on your local laptop or desktop computer.

<!--more-->


### Download CRC

```
wget https://mirror.openshift.com/pub/openshift-v4/clients/crc/latest/crc-linux-amd64.tar.xz
tar xvf crc-linux-amd64.tar.xz
sudo cp crc-linux-*-amd64/crc /usr/local/sbin/
crc version
```

### Setup CRS
```
crc setup
```

```
INFO Checking if running as non-root
INFO Caching oc binary
INFO Setting up virtualization
INFO Setting up KVM
INFO Installing libvirt service and dependencies
INFO Adding user to libvirt group
INFO Enabling libvirt
INFO Starting libvirt service
INFO Will use root access: start libvirtd service
[sudo] password for devopstales:
INFO Checking if a supported libvirt version is installed
INFO Installing crc-driver-libvirt
INFO Removing older system-wide crc-driver-libvirt
INFO Setting up libvirt 'crc' network
INFO Starting libvirt 'crc' network
INFO Checking if NetworkManager is installed
INFO Checking if NetworkManager service is running
INFO Writing Network Manager config for crc
INFO Will use root access: write NetworkManager config in /etc/NetworkManager/conf.d/crc-nm-dnsmasq.conf
INFO Will use root access: execute systemctl daemon-reload command
INFO Will use root access: execute systemctl stop/start command
INFO Writing dnsmasq config for crc
INFO Will use root access: write dnsmasq configuration in /etc/NetworkManager/dnsmasq.d/crc.conf
INFO Will use root access: execute systemctl daemon-reload command
INFO Will use root access: execute systemctl stop/start command
INFO Unpacking bundle from the CRC binary
Setup is complete, you can now run 'crc start' to start the OpenShift cluster
```

Thec configuration creats two dnsmasq config:
```
cat /etc/NetworkManager/conf.d/crc-nm-dnsmasq.conf
[main]
dns=dnsmasq

cat /etc/NetworkManager/dnsmasq.d/crc.conf
server=/apps-crc.testing/192.168.130.11
server=/crc.testing/192.168.130.11
```

The first enable the NetworkManager's dnsmasq plugin to be used as a dns server and the second points two dns zone the `*.apps-crc.testing` and the `*.crc.testing` to the ip of the new vm's ip. The crc creats a kvm network for the vm whit this ip range.

```
sudo virsh net-list
 Name      State    Autostart   Persistent
--------------------------------------------
 crc       active   yes         yes
 default   active   yes         yes
```

```
sudo virsh net-edit
 <network>
  <name>crc</name>
  <uuid>49eee855-d342-46c3-9ed3-b8d1758814cd</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='crc' stp='on' delay='0'/>

  <mac address='52:54:00:fd:be:d0'/>
  <ip family='ipv4' address='192.168.130.1' prefix='24'>
    <dhcp>
      <host mac='52:fd:fc:07:21:82' ip='192.168.130.11'/>
    </dhcp>
  </ip>
</network>
```

### Start CRS
```
crc start
```

```
INFO Checking if running as non-root
INFO Checking if oc binary is cached
INFO Checking if Virtualization is enabled
INFO Checking if KVM is enabled
INFO Checking if libvirt is installed
INFO Checking if user is part of libvirt group
INFO Checking if libvirt is enabled
INFO Checking if libvirt daemon is running
INFO Checking if a supported libvirt version is installed
INFO Checking if crc-driver-libvirt is installed
INFO Checking if libvirt 'crc' network is available
INFO Checking if libvirt 'crc' network is active
INFO Checking if NetworkManager is installed
INFO Checking if NetworkManager service is running
INFO Checking if /etc/NetworkManager/conf.d/crc-nm-dnsmasq.conf exists
INFO Checking if /etc/NetworkManager/dnsmasq.d/crc.conf exists
? Image pull secret [? for help]
```
Please note that a valid OpenShift user pull secret is required during installation. The pull secret can be copied or downloaded from the Pull Secret section of the [Install on Laptop: Red Hat CodeReady Containers](https://cloud.redhat.com/openshift/install/crc/installer-provisioned) page on cloud.redhat.com.

```
INFO To access the cluster, first set up your environment by following 'crc oc-env' instructions
INFO Then you can access it by running 'oc login -u developer -p developer https://api.crc.testing:6443'
INFO To login as an admin, run 'oc login -u kubeadmin -p 7z6T5-qmTth-oxaoD-p3xQF https://api.crc.testing:6443'
INFO
INFO You can now run 'crc console' and use these credentials to access the OpenShift web console
```

Go to the console:
```
crc console
```

![Example image](/img/include/okd4.webp)

