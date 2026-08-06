---
title: Install pxeboot server
url: https://devopstales.github.io/linux/pxe1/
date: 2019-07-16
---


Have you ever needed to troubleshoot or diagnose a problematic computer and you forgot where the utility CD is? We’ll show you how to utilize network booting (PXE) to make that problem a thing of the past.
<!--more-->

### dhcp

```
yum install -y dhcp nano

# ha több interfész van
echo "DHCPDARGS=enp0s8" >> /etc/sysconfig/dhcpd

cat > /etc/dhcp/dhcpd.conf << EOF
#DHCP configuration for PXE boot server
ddns-update-style interim;
ignore client-updates;
authoritative;
allow booting;
allow bootp;
allow unknown-clients;

# A slightly different configuration for an internal subnet.
subnet 192.168.100.0
netmask 255.255.255.0
{
range 192.168.100.101 192.168.100.200;
option domain-name-servers 192.168.100.100;
option routers 192.168.100.100;
default-lease-time 600;
max-lease-time 7200;

# PXE SERVER IP
next-server 192.168.100.100; #  DHCP server ip
filename "pxelinux.0";
}
EOF

systemctl start dhcpd.service
systemctl enable dhcpd.service
firewall-cmd --permanent --add-service={dhcp,proxy-dhcp}
firewall-cmd --reload
```

```
cd /opt
wget http://ftp.bme.hu/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso
wget http://ftp.bme.hu/centos/7/isos/x86_64/sha256sum.txt

sha256sum CentOS-7-x86_64-Minimal-1810.iso
cat sha256sum.txt

mount -o loop /opt/CentOS-7-x86_64-Minimal-1810.iso  /mnt
```

### ftp

```
yum install -y tftp-server vsftpd syslinux syslinux-tftpboot

cp -v /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/
cp -v /usr/share/syslinux/menu.c32 /var/lib/tftpboot/
cp -v /usr/share/syslinux/mboot.c32 /var/lib/tftpboot/
cp -v /usr/share/syslinux/chain.c32 /var/lib/tftpboot/

mkdir /var/lib/tftpboot/pxelinux.cfg
mkdir -p /var/lib/tftpboot/networkboot/rhel7
mkdir /var/ftp/pub/rhel7

cp -raf /mnt/* /var/ftp/pub/rhel7
cp /var/ftp/pub/rhel7/images/pxeboot/{initrd.img,vmlinuz} /var/lib/tftpboot/networkboot/rhel7/
umount /mnt
```

```
cat > /var/lib/tftpboot/pxelinux.cfg/default << EOF
default menu.c32
prompt 0
timeout 30
menu title mydomail.local PXE Menu

label 1
menu label ^1) Boot from local drive
localboot 0x00

label 2
menu label ^2) CentOS 7 X64
kernel /networkboot/rhel7/vmlinuz
append initrd=/networkboot/rhel7/initrd.img ks=ftp://192.168.100.100/pub/rhel7/ks.cfg inst.repo=ftp://192.168.100.100/pub/rhel7 devfs=nomount

label 3
menu label ^3) CentOS 7 X64
kernel /networkboot/rhel7/vmlinuz
append initrd=/networkboot/rhel7/initrd.img inst.repo=ftp://192.168.100.100/pub/rhel7 devfs=nomount

#label 4
#menu label ^4) Install CentOS 7 x64 with Local Repo using VNC
#kernel /networkboot/rhel7/vmlinuz
#append  initrd=/networkboot/rhel7/initrd.img inst.repo=ftp://192.168.100.100/pub/rhel7 devfs=nomount inst.vnc inst.vncpassword=Aa123456
EOF
```

```
echo '# Use network installation
url --url="ftp://192.168.100.100/pub/rhel7/"
install

firstboot --disable
selinux --disabled
firewall --disabled
services --enabled=NetworkManager,sshd,chronyd
eula --agreed
ignoredisk --only-use=sda
# Keyboard layouts
keyboard --vckeymap=hu --xlayouts='hu'
# System language
lang en_US.UTF-8

#Text mode installation
text

# Network information
network  --bootproto=dhcp --device=eth0 --hostname=test.mydomain --noipv6 --activate
# Root password
authconfig --enableshadow --enablemd5
# Gen hash: openssl passwd -1 "Password1"
rootpw --iscrypted $1$Skulk114$/QcbiPYHtUnB/rJaAehAH0
# System timezone
timezone Europe/Budapest --ntpservers=0.hu.pool.ntp.org,.hu.pool.ntp.org,2.hu.pool.ntp.org

# System bootloader configuration
bootloader --append="net.ifnames=0 biosdevname=0" --location=mbr
zerombr

# Disk partitioning information
clearpart --all --drives=sda
part /boot --fstype=ext4 --size=512
part pv.01 --size=1 --grow

volgroup vg00 pv.01
logvol swap --name=lv_swap --vgname=vg00 --size=1024
logvol / --name=lv_01 --vgname=vg00 --size=1 --grow

reboot --eject

%packages
@core
chrony
openssh-clients
net-tools
%end' > /var/ftp/pub/rhel7/ks.cfg
```

```
systemctl start tftp.service
systemctl enable tftp.service
systemctl enable vsftpd.service
systemctl start vsftpd.service

firewall-cmd --permanent --add-service=tftp
firewall-cmd --permanent --add-service=ftp
firewall-cmd --reload
```

CLIENT NEEDS AT LEASET 2 GB RAM

