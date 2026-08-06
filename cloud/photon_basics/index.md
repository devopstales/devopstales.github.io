---
title: PhotonOS Basics
url: https://devopstales.github.io/cloud/photon_basics/
date: 2020-03-04
keywords: Vagrant, VMware, PhotonOS
---


Project Photon OS is an open source, minimal Linux container host that is optimized for cloud-native applications, cloud platforms, and VMware infrastructure.

<!--more-->

### Star in Vagrant
```
nano Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "devopstales/photon3"
  config.vm.synced_folder ".", "/vagrant", type: "virtualbox"

  config.timezone.value = :host
  config.vm.provider "virtualbox" do |vb|
    vb.name = "photon01"
      vb.memory = 4096

      vb.cpus = 2
      vb.linked_clone = true
      vb.customize ["modifyvm", :id, "--vram", "8"]
  end

  config.vm.define "pothon01" do |vbox|
    vbox.vm.network :public_network, ip: "192.168.0.112" , bridge: "wlan0" #, adapter: "1"
    vbox.vm.hostname = "pothon01.mydomain.intra"
    vbox.hostsupdater.remove_on_suspend = false
    vbox.vbguest.auto_update = false
  end
end
```

```
vagrant up
vagrant ssh
```

When you try to login the server wants you to change the passowrd of the `vagrant` user. The base password is `vagrant` as usual.

```
vagrant ssh
You are required to change your password immediately (password expired)
Last login: Sun Apr 14 20:30:22 2019 from 10.0.2.2
WARNING: Your password has expired.
You must change your password now and login again!
Changing password for vagrant.
Current password:
New password:
Retype new password:
passwd: password updated successfully
Connection to 127.0.0.1 closed.
```

Vagrant can't configure the ip, hostname and mount the `/vagrant` fonder.

### Package management
On Photon OS, `tdnf` is the default package manager for installing new packages. It is a C implementation of the `DNF` package manager without Python dependencies. `DNF` is the next upcoming major version of yum.

Let's install packages for the next steps.
```
tdnf install nano awk tar build-essential linux-devel less -y
```

### Install virtualbox guest additions

![Example image](/img/include/photon_base_1.webp)<br><br>

```
mount /dev/cdrom /mnt/cdrom
cd /mnt/cdrom
./VBoxLinuxAdditions.run
```

### Configure static ip
PhotonOS use systemd-networkd to manage network configurations. systemd-networkd configorations is located under `/etc/systemd/network/`.
```
cat /etc/systemd/network/99-dhcp-en.network
[Match]
Name=e*

[Network]
DHCP=yes
IPv6AcceptRA=no
```

```
cat > /etc/systemd/network/20-static-eth1.network << "EOF"
[Match]
Name=eth1

[Network]
DHCP=no
Address=192.168.0.112/24
Gateway=192.168.0.1
DNS=8.8.8.8
Domains=mydomain.intra
NTP=0.pool.ntp.org
EOF
```

```
chmod 644 /etc/systemd/network/20-static-eth1.network
systemctl restart systemd-networkd
systemctl status systemd-networkd -l
```

### Configure hostname
```
hostnamectl set-hostname "pothon01.mydomain.intra"
```

```
hostnamectl status
   Static hostname: pothon01.mydomain.intra
         Icon name: computer-vm
           Chassis: vm
        Machine ID: 2c230d2255834f75bff4872bff234df4
           Boot ID: 2708cffec4cf4c6e91053cc11825b590
    Virtualization: oracle
  Operating System: VMware Photon OS/Linux
            Kernel: Linux 4.19.32-3.ph3
      Architecture: x86-64
```

