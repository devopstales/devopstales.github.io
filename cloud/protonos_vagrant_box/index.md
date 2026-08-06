---
title: How to create Vagrant box?
url: https://devopstales.github.io/cloud/protonos_vagrant_box/
date: 2020-03-03
keywords: Vagrant, VMware, PhotonOS
---


In this post I will show you how to create a vagrant box from pothonos ISO.

<!--more-->

### Base VM
Fist we need to create a virtualbox vm and install the [latest PothonOS ISO](https://github.com/vmware/photon/wiki/Downloading-Photon-OS).

* Name: Photon-base
* Type: Linux
* Version: Other Linux (64-bit)
* Memory Size: 1024MB
* New Virtual Disk: [Type: VDI, Size: 40 GB]

Modify the hardware settings of the virtual machine:

* Disable audio
* Disable USB
* Ensure Network Adapter 1 is set to NAT
* Add this port-forwarding rule:
 * Name: SSH
 * Protocol: TCP
 * Host IP: blank
 * Host Port: 2222
 * Guest IP: blank
 * Guest Port: 22

### Configure System

Change the root users password to `vagrant`
```
passwd
chage -I -1 -m 0 -M 99999 -E -1 root
```

Upgrade system and install packages:
```
tdnf upgrade -y
tdnf install sudo nano wget awk tar build-essential linux-devel less -y
```

### Create the vagrant account
Next you need to create the default vagrant user account:
```
useradd -m -G sudo vagrant
passwd vagrant
chage -I -1 -m 0 -M 99999 -E -1 vagrant

cat > /etc/sudoers.d/vagrant < EOF
# add vagrant user
vagrant ALL=(ALL) NOPASSWD:ALL
EOF
```

### Change ssh config
```
nano /etc/ssh/sshd_config
AuthorizedKeysFile     %h/.ssh/authorized_keys
```

```
systemctl restart sshd
```

### Add vagrant ssh keys to vagrant user
```
su - vagrant

mkdir -p /home/vagrant/.ssh
chmod 0700 /home/vagrant/.ssh
```

```
wget --no-check-certificate \
    https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub \
    -O /home/vagrant/.ssh/authorized_keys
```

```
chmod 0600 /home/vagrant/.ssh/authorized_keys
chown –R vagrant /home/vagrant/.ssh
```

### Add VirtualBoxadditions

Go to your virtualbox menu for the VM and select `Devices / Insert Guest Additions CD Image`

![Example image](/img/include/photon_base_1.webp)<br><br>

```
mount /dev/cdrom /mnt/cdrom
cd /mnt/cdrom
./VBoxLinuxAdditions.run
```

### Compress vm
```
sudo dd if=/dev/zero of=/EMPTY bs=1M
sudo rm -f /EMPTY
```

### Package box
```
vagrant package --base "Photon-base"

vagrant box add <user-name>/photon3 package.box
vagrant box add devopstales/photon3 package.box
```

