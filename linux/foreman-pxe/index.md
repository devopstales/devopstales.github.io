---
title: Install Foreman PXE boot
url: https://devopstales.github.io/linux/foreman-pxe/
date: 2019-11-07
keywords: foreman, the foreman, katello, PXE, PXE boot
---


Foreman is a complete lifecycle management tool for physical and virtual servers. We give system administrators the power to easily automate repetitive tasks, quickly deploy applications, and proactively manage servers, on-premise or in the cloud.
<!--more-->

I hawe a VM with two virtual interface the enp0s3 for NAT and enp0s9 with an internal network.


### Install DHCP server
```
yum install -y dhcp nano -y

echo "DHCPDARGS=enp0s9" >> /etc/sysconfig/dhcpd
```

```
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
next-server 192.168.100.100; #  PXE server ip
filename "pxelinux.0";
}
EOF
```

```
systemctl start dhcpd.service
systemctl enable dhcpd.service
systemctl status dhcpd.service
```

### Install DNS server
```
yum -y install bind bind-utils

nano /etc/named.conf
options {
        ...
        // listen-on port 53 { 127.0.0.1; };
        // listen-on-v6 port 53 { ::1; };
        ...
        allow-query     { localhost; 192.168.100.0/24; };
        ...
        forwarders {
                8.8.8.8;
                8.8.4.4;
        };
};
include "/etc/named.my.zones";
```

```
touch /etc/named.my.zones
chown root:named /etc/named.my.zones

nano /etc/named.my.zones
zone "devopstales.intra" IN {
         type master;
         file "devopstales.intra.db";
         allow-update { none; };
};
```

```
nano /var/named/devopstales.intra.db
@   IN  SOA     primary.devopstales.intra. root.mydomain.intra. (
                                                1001    ;Serial
                                                3H      ;Refresh
                                                15M     ;Retry
                                                1W      ;Expire
                                                1D      ;Minimum TTL
                                                )

;Name Server Information
@      IN  NS      primary.devopstales.intra.

;IP address of Name Server
primary IN  A       192.168.100.100

;Mail exchanger
devopstales.intra. IN  MX 10   mail.mydomain.intra.

;A - Record HostName To IP Address
foreman IN  A       192.168.100.100
mail    IN  A       192.168.100.50
```

```
systemctl restart named
systemctl status named
systemctl enable named
```

### Install vtftp
```
yum install vsftpd -y

nano /etc/vsftpd/vsftpd.conf
anonymous_enable=YES
write_enable=NO

systemctl enable vsftpd
systemctl restart vsftpd
systemctl status vsftpd

cd /opt
wget http://ftp.bme.hu/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1908.iso
wget http://ftp.bme.hu/centos/7/isos/x86_64/sha256sum.txt

sha256sum CentOS-7-x86_64-Minimal-1908.iso
cat sha256sum.txt

mount -o loop /opt/CentOS-7-x86_64-Minimal-1908.iso  /mnt

mkdir /var/ftp/pub/CentOS_7_x86_64
rsync -rv --progress /mnt/ /var/ftp/pub/CentOS_7_x86_64/
umount /mnt
restorecon -Rv /var/ftp/pub/
```

### Install Foreman
```
yum -y install https://yum.puppet.com/puppet6-release-el-7.noarch.rpm
yum -y install http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum -y install https://yum.theforeman.org/releases/1.23/el7/x86_64/foreman-release.rpm
yum -y install foreman-installer

foreman-installer \
--foreman-initial-organization "mydomain" \
--foreman-initial-location "office" \
--enable-foreman-plugin-ansible \
--enable-foreman-proxy-plugin-ansible \
--enable-foreman-plugin-remote-execution \
--enable-foreman-proxy-plugin-remote-execution-ssh \
--enable-foreman-plugin-cockpit \
--enable-foreman-plugin-openscap
```

### Configure hammer
```
nano ~/.hammer/cli.modules.d/foreman.yml
:foreman:
 :host: 'https://foreman.devopstales.intra/'
 :username: 'admin'
 :password: '**********'

hammer defaults add --param-name organization --param-value "mydomain"
hammer defaults add --param-name location --param-value "office"
hammer defaults list
```

### Configurate PXEboot
```
sudo ss -lnup | grep 69
grep disa /etc/xinetd.d/tftp
ls -l /var/lib/tftpboot/

# create subnet
hammer subnet create \
--name PXEnet \
--network-type IPv4 \
--network 192.168.100.0 \
--mask 255.255.255.0 \
--dns-primary 192.168.100.100 \
--domains devopstales.intra \
--tftp-id 1 \
--httpboot-id 1 \
--ipam "Internal DB" \
--from 192.168.100.101 \
--to 192.168.100.200 \
--boot-mode Static

hammer medium create \
--name "CentOS7_DVD_FTP" \
--os-family "Redhat" \
--path "ftp://foreman.devopstales.intra/pub/CentOS_7_x86_64/"
```

Create a file hardened_ptable.txt with the content below.

```
<%#
kind: ptable
name: Kickstart hardened
oses:
- CentOS
- Fedora
- RedHat
%>

# System bootloader configuration
bootloader --location=mbr --boot-drive=sda --timeout=3
# Partition clearing information
clearpart --all --drives=sda
zerombr

# Disk partitioning information
part /boot --fstype="xfs" --ondisk=sda --size=1024 --label=boot --fsoptions="rw,nodev,noexec,nosuid"

# 30GB physical volume
part pv.01  --fstype="lvmpv" --ondisk=sda --size=30720
volgroup vg_os pv.01

logvol /        --fstype="xfs"  --size=4096 --vgname=vg_os --name=lv_root
logvol /home    --fstype="xfs"  --size=512  --vgname=vg_os --name=lv_home --fsoptions="rw,nodev,nosuid"
logvol /tmp     --fstype="xfs"  --size=1024 --vgname=vg_os --name=lv_tmp  --fsoptions="rw,nodev,noexec,nosuid"
logvol /var     --fstype="xfs"  --size=6144 --vgname=vg_os --name=lv_var  --fsoptions="rw,nosuid"
logvol /var/log --fstype="xfs"  --size=512  --vgname=vg_os --name=lv_log  --fsoptions="rw,nodev,noexec,nosuid"
logvol swap     --fstype="swap" --size=2048 --vgname=vg_os --name=lv_swap --fsoptions="swap"
```

```
hammer partition-table create \
  --name "Kickstart hardened" \
  --os-family "Redhat" \
  --operatingsystems "CentOS 7.4.1708" \
  --file "hardened_ptable.txt"

hammer os create \
  --name "CentOS" \
  --major "7" \
  --minor "4.1708" \
  --family "Redhat" \
  --password-hash "SHA512" \
  --architectures "x86_64" \
  --media "CentOS7_DVD_FTP" \
  --partition-tables "Kickstart hardened"

hammer hostgroup create \
  --name "el7_group" \
  --description "Host group for CentOS 7 servers" \
  --lifecycle-environment "stable" \
  --content-view "el7_content" \
  --content-source-id "1" \
  --environment "homelab" \
  --puppet-proxy "foreman.devopstales.intra" \
  --puppet-ca-proxy "foreman.devopstales.intra" \
  --domain "devopstales.intra" \
  --subnet "PXEnet" \
  --architecture "x86_64" \
  --operatingsystem "CentOS 4.1708" \
  --medium "CentOS7_DVD_FTP" \
  --partition-table "Kickstart hardened" \
  --pxe-loader "PXELinux BIOS" \
  --root-pass "Password1"

hammer hostgroup set-parameter  \
  --name "selinux-mode" \
  --value "disabled" \
  --hostgroup "el7_group"

hammer hostgroup set-parameter  \
  --name "disable-firewall" \
  --value "true" \
  --hostgroup "el7_group"

hammer hostgroup set-parameter  \
  --name "bootloader-append" \
  --value "net.ifnames=0 biosdevname=0" \
  --hostgroup "el7_group"

hammer host create \
  --name "pxe-test" \
  --hostgroup "el7_group" \
  --interface "type=interface,mac=08:00:27:fb:ad:17,ip=192.168.100.110,managed=true,primary=true,provision=true"
```

```
ll /var/lib/tftpboot/pxelinux.cfg/
cat /var/lib/tftpboot/pxelinux.cfg/01-08-00-27-fb-ad-17
```

