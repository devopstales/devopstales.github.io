---
title: Install Rundeck
url: https://devopstales.github.io/linux/rundeck/
date: 2019-04-09
---


Rundeck is open source software that powers self-service operations.

<!--more-->

### Install MySQL
```
echo "[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.0/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1" > /etc/yum.repos.d/MariaDB.repo

yum -y install httpd MariaDB-Galera-server.x86_64

systemctl enable httpd
systemctl enable mysql
systemctl start httpd
systemctl start mysql

mysql_secure_installation
mysql -u root -p
create database rundeck;
grant all on rundeck.* to 'rundeck'@'localhost' identified by 'Password1';
quit
```

### Install rundeck
```
yum install java-1.8.0 httpd -y
rpm -Uvh http://repo.rundeck.org/latest.rpm
yum install rundeck
```

### Configure rundeck
```
nano /etc/rundeck/rundeck-config.properties
# change hostname here
# grails.serverURL=http://localhost:4440
grails.serverURL=http://rundeck.devopstales.intra
#dataSource.url = jdbc:h2:file:/var/lib/rundeck/data/rundeckdb
dataSource.url = jdbc:mysql://localhost/rundeck?autoReconnect=true&useSSL=false
dataSource.username=rundeck
dataSource.password=Password1
dataSource.driverClassName=com.mysql.jdbc.Driver
# mail server
grails.mail.host=localhost
grails.mail.port=25

service rundeckd start
```

### Configure httpd
```
nano /etc/httpd/conf.d/rundeck_proxy.conf
<virtualhost *:80>
        ServerName rundeck.devopstales.intra
        ServerAlias www.rundeck.devopstales.intra
        ServerAdmin admin@rundeck.devopstales.intra

        ProxyRequests Off
        ProxyPass / http://localhost:4440/
        ProxyPassReverse / http://localhost:4440/
</virtualhost>

service httpd restart
```

