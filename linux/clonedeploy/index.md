---
title: Install clonedeploy pxeboot server
url: https://devopstales.github.io/linux/clonedeploy/
date: 2019-07-17
keywords: PXE, clonedeploy
---


I’ll show you how to create network booting (PXE) with clusterdeploy.
<!--more-->

### Install Web Application
```
yum -y install yum-utils
yum -y install epel-release
rpm --import "http://keyserver.ubuntu.com/pks/lookup?op=get&search=0x3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF"
yum-config-manager --add-repo http://download.mono-project.com/repo/centos7/

yum -y install mono-devel apache2-mod_mono httpd udpcast lz4 mkisofs wget
```

```
wget "https://sourceforge.net/projects/clonedeploy/files/CloneDeploy 1.4.0/clonedeploy-1.4.0.tar.gz"
tar xvzf clonedeploy-1.4.0.tar.gz
cd clonedeploy
cp clonedeploy.conf /etc/httpd/conf.d/
mkdir /var/www/html/clonedeploy
cp -r frontend /var/www/html/clonedeploy
cp -r api /var/www/html/clonedeploy
cp -r tftpboot /

ln -s ../../images /tftpboot/proxy/bios/images
ln -s ../../images /tftpboot/proxy/efi32/images
ln -s ../../images /tftpboot/proxy/efi64/images
ln -s ../../kernels /tftpboot/proxy/bios/kernels
ln -s ../../kernels /tftpboot/proxy/efi32/kernels
ln -s ../../kernels /tftpboot/proxy/efi64/kernels

mkdir -p /cd_dp/images
mkdir /cd_dp/resources
mkdir /var/www/.mono
mkdir /usr/share/httpd/.mono
mkdir /etc/mono/registry
chown -R apache:apache /tftpboot /cd_dp /var/www/html/clonedeploy /var/www/.mono /usr/share/httpd/.mono /etc/mono/registry
chmod 1777 /tmp

sysctl fs.inotify.max_user_instances=1024
echo fs.inotify.max_user_instances=1024 >> /etc/sysctl.conf
chkconfig httpd on
```

### Install Database
```
echo "[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.3/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1" >> /etc/yum.repos.d/MariaDB.repo

yum -y install mariadb-server
service mariadb start
chkconfig mariadb on

mysql

create database clonedeploy;
CREATE USER 'cduser'@'localhost' IDENTIFIED BY 'Password1';
GRANT ALL PRIVILEGES ON clonedeploy.* TO 'cduser'@'localhost';
quit

mysql clonedeploy < cd.sql -v
```

Open `/var/www/html/clonedeploy/api/Web.config` with a text editor and change the following values:<br>
`xx_marker1_xx` to your cduser database password you created earlier<br>
On that same line change `Uid=root to Uid=cduser`<br>
`xx_marker2_xx` to some random characters(alphanumeric only), probably should be a minimum of 8<br>

```
service httpd restart
```

### Install Samba Server

```
yum -y install samba
groupadd cdsharewriters
useradd cd_share_ro
useradd cd_share_rw -G cdsharewriters
usermod -a -G cdsharewriters apache

smbpasswd -a cd_share_ro
smbpasswd -a cd_share_rw

echo "[cd_share]
path = /cd_dp
valid users = @cdsharewriters, cd_share_ro
create mask = 02775
directory mask = 02775
guest ok = no
writable = yes
browsable = yes
read list = @cdsharewriters, cd_share_ro
write list = @cdsharewriters
force create mode = 02775
force directory mode = 02775
force group = +cdsharewriters" >> /etc/samba/smb.conf

chown -R apache:cdsharewriters /cd_dp
chmod -R 2775 /cd_dp
service smb restart
chkconfig smb on
```

### Install TFTP Server
```
yum -y install tftp-server
sed -i 's/\/var\/lib\/tftpboot/\/tftpboot -m \/tftpboot\/remap/g' /usr/lib/systemd/system/tftp.service
systemctl daemon-reload
service tftp restart
chkconfig tftp on
```

### Create Firewall Exceptions
```
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-service=samba
firewall-cmd --permanent --add-service=tftp
firewall-cmd --permanent --add-service=dhcp
firewall-cmd --permanent --add-service=proxy-dhcp
firewall-cmd --permanent --add-port=9000-10002/udp
service firewalld reload
```

### Post Install Setup
```
# Open the CloneDeploy Web Interface
http://server-ip/clonedeploy

# éogin with
clonedeploy / password

# Upon login you will be greeted with the Initial Setup Page
# Fill out the fields and click Finalize Setup
```


```
yum -y install syslinux vsftpd
cp -v /usr/share/syslinux/pxelinux.0 /tftpboot/
cp -v /usr/share/syslinux/vesamenu.c32 /tftpboot/

mkdir /var/ftp/pub/rhel7
mkdir -p /tftpboot/networkboot/rhel7/

cd /opt
wget  http://files.clonedeploy.org/CloneDeploy-Services.dll
cp CloneDeploy-Services.dll /var/www/html/clonedeploy/api/bin/

wget http://ftp.bme.hu/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso
wget http://ftp.bme.hu/centos/7/isos/x86_64/sha256sum.txt

sha256sum CentOS-7-x86_64-Minimal-1810.iso
cat sha256sum.txt

mount -o loop /opt/CentOS-7-x86_64-Minimal-1810.iso  /mnt
cp -raf /mnt/* /var/ftp/pub/rhel7
cp /var/ftp/pub/rhel7/images/pxeboot/{initrd.img,vmlinuz} /tftpboot/networkboot/rhel7/

systemctl enable vsftpd.service && systemctl start vsftpd.service

LABEL CentOS 7 X64
MENU LABEL CentOS 7 X64
kernel /networkboot/rhel7/vmlinuz
append initrd=/networkboot/rhel7/initrd.img inst.repo=ftp://192.168.100.100/pub/rhel7 devfs=nomount
```

---
http://clonedeploy.org/docs/create-and-deploy-your-first-image/0

