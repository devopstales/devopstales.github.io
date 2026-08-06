---
title: Proxmox backup with znapzend
url: https://devopstales.github.io/virtualization/proxmox-backup-znapzend/
date: 2019-03-11
keywords: proxmox ve, proxmox cluster, znapzendzetup, znapzend setup, znapzend example
---


Backup your machines on Pfsense with znapzend tool.

<!--more-->

### The servers
```
proxmox-node1 192.168.10.20
omv-nas1 192.168.10.50
```

### Create sub volume
Note that while recursive configurations are well supported to set up backup and retention policies for a whole dataset subtree under the dataset to which you have applied explicit configuration, at this time pruning of such trees ("I want every dataset under var except var/tmp") is not supported.

```
zfs create local-zfs/vm-data
pvesm add zfspool local-zfs --pool local-zfs/vm-data
```

### Build znapzend on servers
```
apt-get install perl unzip git mbuffer build-essential git

cd /root
git clone https://github.com/oetiker/znapzend
cd /root/znapzend
./configure --prefix=/opt/znapzend

make
make install

ln -s /opt/znapzend/bin/znapzend /usr/local/bin/znapzend
ln -s /opt/znapzend/bin/znapzendzetup /usr/local/bin/znapzendzetup
ln -s /opt/znapzend/bin/znapzendztatz /usr/local/bin/znapzendztatz

znapzend --version
```

Or you can download a the deb package from here:
https://github.com/devopstales/znapzend-debian/releases

### Install znapzend on servers
```
dpkg -i znapzend_0.19.1_amd64_stretch.deb
```

### Configure znapzend
```
znapzendzetup create --recursive\
--mbuffer=/usr/bin/mbuffer \
--mbuffersize=1G \
SRC '2d=>1d' local-zfs/vmdata \
DST:a '14d=>1d' root@192.168.10.50:tank

# test
znapzend --debug --noaction --runonce=local-zfs
znapzendzetup list
```

### Create znapzend service
```
nano /etc/default/znapzend
ZNAPZENDOPTIONS="--logto=/var/log/znapzend.log"

systemctl enable znapzend.service
systemctl restart znapzend.service
systemctl status znapzend.service
```

