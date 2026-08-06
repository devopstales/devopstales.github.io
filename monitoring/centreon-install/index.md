---
title: Install Centreon on Centos 7
url: https://devopstales.github.io/monitoring/centreon-install/
date: 2019-04-27
keywords: centreon, nagios
---


Centreon is a free Open Source monitoring software which allows an administrator to easily configure alerts based on thresholds, generate email alerts, add systems to be monitored quickly without the need of configuring complicated configuration files.

<!--more-->

### Install requisites
```
selinuxenabled && echo enabled || echo disabled

nano /etc/selinux/config
SELINUX=disabled

yum install -y epel-release mariadb-server
mkdir /etc/systemd/system/mariadb.service.d/
echo -ne "[Service]\nLimitNOFILE=32000\n" | tee /etc/systemd/system/mariadb.service.d/limits.conf

systemctl daemon-reload
systemctl enable mariadb
systemctl start mariadb
systemctl status mariadb

yum install centos-release-scl
```

### Install Centreon
```
cd /opt
wget http://yum.centreon.com/standard/19.04/el7/stable/noarch/RPMS/centreon-release-19.04-1.el7.centos.noarch.rpm
yum install --nogpgcheck centreon-release-19.04-1.el7.centos.noarch.rpm
yum install -y centreon-base-config-centreon-engine centreon

echo "date.timezone = Europe/Budapest" > /etc/opt/rh/rh-php71/php.d/php-timezone.ini
systemctl restart rh-php71-php-fpm

systemctl enable httpd24-httpd
systemctl enable snmpd
systemctl enable snmptrapd
systemctl enable rh-php71-php-fpm
systemctl enable centcore
systemctl enable centreontrapd
systemctl enable cbd
systemctl enable centengine
systemctl enable centreon

systemctl start rh-php71-php-fpm
systemctl start httpd24-httpd
systemctl start mysqld
systemctl start cbd
systemctl start snmpd
systemctl start snmptrapd
```

### Configurate centreon

![Example image](/img/include/centreon_1.webp)
![Example image](/img/include/centreon_2.webp)
![Example image](/img/include/centreon_3.webp)
![Example image](/img/include/centreon_4.webp)
![Example image](/img/include/centreon_5.webp)
![Example image](/img/include/centreon_6.webp)
![Example image](/img/include/centreon_7.webp)
![Example image](/img/include/centreon_8.webp)
![Example image](/img/include/centreon_9.webp)

```
systemctl start cbd
systemctl start centcore
systemctl start centreontrapd
yum install centreon-widget* -y
```

![Example image](/img/include/centreon_10.webp)
Select Central and Click Export Configuration. <br>
Then the poller will ativated.
![Example image](/img/include/centreon_11.webp)

