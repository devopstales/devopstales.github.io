---
title: Proxmox backup with sanoid
url: https://devopstales.github.io/virtualization/proxmox-backup-sanoid/
date: 2019-03-11
keywords: proxmox ve, proxmox cluster
---


Backup your machines on Pfsense with sanoid tool.

<!--more-->

### The servers
```
proxmox-node1 192.168.10.20
omv-nas1 192.168.10.50
```

### Build sanoid on servers
```
apt-get install libcapture-tiny-perl libconfig-inifiles-perl git

cd /opt
git clone https://github.com/jimsalterjrs/sanoid

ln /opt/sanoid/sanoid /usr/sbin/
```

Or you can build deb package:

### Build and install sanoid deb package
```
sudo apt-get install devscripts debhelper dh-systemd
git clone https://github.com/jimsalterjrs/sanoid.git
cd sanoid
debuild -us -uc

cd ..
sudo apt-get install libconfig-inifiles-perl
sudo dpkg -i sanoid_2.0.1_all.deb
```

### Configure sanoid
```
mkdir -p /etc/sanoid
cp /opt/sanoid/sanoid.conf /etc/sanoid/sanoid.conf
cp /opt/sanoid/sanoid.defaults.conf /etc/sanoid/sanoid.defaults.conf

nano /etc/crontab
* 2 * * * root /usr/sbin/sanoid --cron
* 3 * * * root /usr/sbin/syncoid --recursive tank root@192.168.10.50:tank

```

```
    ####################
    # sanoid.conf file #
    ####################
    [local-zfs]
            use_template = production
    #############################
    # templates below this line #
    #############################
    [template_production]
            # store hourly snapshots 36h
            # hourly = 36
            # store 14 days of daily snaps
            daily = 14
            # store back 6 months of monthly
            # monthly = 6
            # store back 3 yearly (remove manually if to large)
            # yearly = 3
            # create new snapshots
            autosnap = yes
            # clean old snapshot
            autoprune = yes
```

