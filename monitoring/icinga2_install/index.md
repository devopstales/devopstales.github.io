---
title: Install Icinga2
url: https://devopstales.github.io/monitoring/icinga2_install/
date: 2019-12-10
keywords: icinga, icinga2, nagios
---


In this tutorial I will show you how to install Icinga2 and Icingaweb2 webinterface.

<!--more-->
Icinga 2 is an open source, scalable and extensible monitoring tool which checks the availability of your network resources, notifies users of outages, and generates performance data for reporting.

### Configure syslinux and Firewall for the install
```
setsebool -P httpd_can_network_connect_db 1
setsebool -P httpd_can_network_connect 1

firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-service=postgresql
firewall-cmd --permanent --add-port=5665/tcp
firewall-cmd --reload
```

### Install postgresql

```
yum install postgresql-server postgresql
postgresql-setup initdb

nano /var/lib/pgsql/data/pg_hba.conf
# icinga
local   icinga      icinga                            md5
host    icinga      icinga      127.0.0.1/32          md5
host    icinga      icinga      ::1/128               md5
local   icinga_web  icinga_web                        md5
host    icinga_web  icinga_web  127.0.0.1/32          md5
host    icinga_web  icinga_web  ::1/128               md5

# "local" is for Unix domain socket connections only
local   all         all                               ident
# IPv4 local connections:
host    all         all         127.0.0.1/32          ident
# IPv6 local connections:
host    all         all         ::1/128               ident

systemctl enable  --now postgresql-10
```
### Install Icinga2

```
yum install https://packages.icinga.com/epel/icinga-rpm-release-7-latest.noarch.rpm
yum install epel-release

yum install icinga2 icinga2-selinux nagios-plugins-all nano-icinga2

systemctl enable --now httpd

echo 'include "/usr/share/nano/icinga2.nanorc"' >> /etc/nanorc
cp /etc/nanorc ~/.nanorc

yum install icinga2-ido-pgsql

cd /tmp
sudo -u postgres psql -c "CREATE ROLE icinga WITH LOGIN PASSWORD 'icinga'"
sudo -u postgres createdb -O icinga -E UTF8 icinga

export PGPASSWORD=icinga
psql -U icinga -d icinga < /usr/share/icinga2-ido-pgsql/schema/pgsql.sql

icinga2 feature enable ido-pgsql

cat << EOF | sudo tee /etc/icinga2/features-enabled/ido-pgsql.conf
/**
 * The db_ido_pgsql library implements IDO functionality
 * for PostgreSQL.
 */

library "db_ido_pgsql"

object IdoPgsqlConnection "ido-pgsql" {
  user = "icinga",
  password = "icinga",
  host = "localhost",
  database = "icinga"
}
EOF

icinga2 api setup


ecjo '
object ApiUser "icingaweb2" {
  password = "Wijsn8Z9eRs5E25d"
  permissions = [ "status/query", "actions/*", "objects/modify/*", "objects/query/*" ]
}' >> /etc/icinga2/conf.d/api-users.conf

icinga2 feature enable command
```

```
icinga2 node wizard
Welcome to the Icinga 2 Setup Wizard!

We'll guide you through all required configuration details.

Please specify if this is a satellite setup ('n' installs a master setup) [Y/n]: n
Starting the Master setup routine...
Please specifiy the common name (CN) [icinga.devopstales.intra]:
Checking for existing certificates for common name 'icinga.devopstales.intra'...
Certificates not yet generated. Running 'api setup' now.
information/cli: Generating new CA.
information/base: Writing private key to '/var/lib/icinga2/ca/ca.key'.
information/base: Writing X509 certificate to '/var/lib/icinga2/ca/ca.crt'.
information/cli: Generating new CSR in '/etc/icinga2/pki/icinga.devopstales.intra.csr'.
information/base: Writing private key to '/etc/icinga2/pki/icinga.devopstales.intra.key'.
information/base: Writing certificate signing request to '/etc/icinga2/pki/icinga.devopstales.intra.csr'.
information/cli: Signing CSR with CA and writing certificate to '/etc/icinga2/pki/icinga.devopstales.intra.crt'.
information/cli: Copying CA certificate to '/etc/icinga2/pki/ca.crt'.
Generating master configuration for Icinga 2.
information/cli: Adding new ApiUser 'root' in '/etc/icinga2/conf.d/api-users.conf'.
information/cli: Enabling the 'api' feature.
Enabling feature api. Make sure to restart Icinga 2 for these changes to take effect.
information/cli: Dumping config items to file '/etc/icinga2/zones.conf'.
information/cli: Created backup file '/etc/icinga2/zones.conf.orig'.
Please specify the API bind host/port (optional):
Bind Host []: Hit Enter
Bind Port []: Hit Enter
information/cli: Created backup file '/etc/icinga2/features-available/api.conf.orig'.
information/cli: Updating constants.conf.
information/cli: Created backup file '/etc/icinga2/constants.conf.orig'.
information/cli: Updating constants file '/etc/icinga2/constants.conf'.
information/cli: Updating constants file '/etc/icinga2/constants.conf'.
information/cli: Updating constants file '/etc/icinga2/constants.conf'.
Done.

Now restart your Icinga 2 daemon to finish the installation!

systemctl restart icinga2
```

### Install IcingaWeb2

```
yum install centos-release-scl
yum install icingaweb2 icingacli icingaweb2-selinux

yum install httpd
systemctl start httpd.service
systemctl enable httpd.service

icingacli setup config webserver apache

## OR

yum install nginx rh-php71-php-fpm rh-php71-php-pgsql
systemctl enable --now rh-php71-php-fpm.service

## config
icingacli setup config webserver nginx > /etc/nginx/default.d//icinga.conf
systemctl enable --now nginx

sudo -u postgres psql -c "CREATE ROLE icinga_web WITH LOGIN PASSWORD 'icinga_web'"
sudo -u postgres createdb -O icinga_web -E UTF8 icinga_web

icingacli setup token create

# go to
https://icinga.devopstales.intra/icingaweb2/
```

![Example image](/img/include/icingaweb1.webp)

![Example image](/img/include/icingaweb2.webp)

![Example image](/img/include/icingaweb3.webp)

![Example image](/img/include/icingaweb4.webp)

![Example image](/img/include/icingaweb5.webp)

![Example image](/img/include/icingaweb6.webp)

![Example image](/img/include/icingaweb7.webp)

![Example image](/img/include/icingaweb8.webp)

![Example image](/img/include/icingaweb9.webp)

![Example image](/img/include/icingaweb10.webp)

![Example image](/img/include/icingaweb11.webp)

![Example image](/img/include/icingaweb12.webp)

![Example image](/img/include/icingaweb13.webp)

![Example image](/img/include/icingaweb14.webp)

![Example image](/img/include/icingaweb15.webp)

![Example image](/img/include/icingaweb16.webp)

