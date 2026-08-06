---
title: Install privacyIDEA
url: https://devopstales.github.io/linux/privacyidea-install/
date: 2019-04-29
---


privacyIDEA is a Two Factor Authentication System which is multi-tenency- and multi-instance-capable. It is opensource, written in Python and hosted at GitHub.
<!--more-->

### Configure privacyidea repo
```
# base os: Debian 9

apt update
apt install dirmngr -y

nano /etc/apt/sources.list.d/privacyidea.list
deb http://lancelot.netknights.it/community/xenial/stable xenial main

wget https://lancelot.netknights.it/NetKnights-Release.asc
apt-key add NetKnights-Release.asc
```

### Install privacyidea
```
apt update
apt install privacyidea-apache2

ln -s /etc/apache2/mods-available/rewrite.load /etc/apache2/mods-enabled/rewrite.load

nano nano sites-enabled/privacyidea.conf
# enable 80 to 443 redirection
systemctl restart apache2
```

### Create admin user
```
pi-manage admin add admin -e admin@localhost
```

