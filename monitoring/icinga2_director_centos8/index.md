---
title: Install icinga director modules to Icingaweb2 on CentOS/Almlalinux/RockyOS 8
url: https://devopstales.github.io/monitoring/icinga2_director_centos8/
date: 2022-12-19
keywords: icinga, icinga2, nagios
---


In this tutorial I will show you how to install Icingaweb2 module director.

<!--more-->
Icinga irector is designed for those who want to automate their configuration deployment and those who want to grant easy access for there users to the Icinga2 configuration.

### Create db for director

```bash
nano /var/lib/pgsql/data/pg_hba.conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# icinga
local   icinga      icinga                            md5
host    icinga      icinga      127.0.0.1/32          md5
host    icinga      icinga      ::1/128               md5
local   icinga_web  icinga_web                        md5
host    icinga_web  icinga_web  127.0.0.1/32          md5
host    icinga_web  icinga_web  ::1/128               md5
local   director    director                          md5
host    director    director    127.0.0.1/32          md5
host    director    director    ::1/128               md5
```

```bash
systemctl restart postgresql

sudo -u postgres psql -c "CREATE ROLE director WITH LOGIN PASSWORD 'director'"
sudo -u postgres createdb -O director -E UTF-8 director


sudo -u postgres psql director -q -c "CREATE USER director WITH PASSWORD 'CHANGEME';
GRANT ALL PRIVILEGES ON DATABASE director TO director;
CREATE EXTENSION pgcrypto;"

```


### Install Icingaweb2 modules

```bash
mkdir -p /usr/share/icingaweb2/modules

MODULE_NAME=ipl
MODULE_VERSION=$(curl -s https://api.github.com/repos/Icinga/icinga-php-library/releases/latest  | grep tag_name | cut -d '"' -f 4)
REPO="https://github.com/Icinga/icinga-php-library"
MODULES_PATH="/usr/share/icingaweb2/modules"
git clone ${REPO} "${MODULES_PATH}/${MODULE_NAME}" --branch "${MODULE_VERSION}"
icingacli module enable "${MODULE_NAME}"

MODULE_NAME=incubator
MODULE_VERSION=$(curl -s https://api.github.com/repos/Icinga/icingaweb2-module-incubator/releases/latest  | grep tag_name | cut -d '"' -f 4)
REPO="https://github.com/Icinga/icingaweb2-module-${MODULE_NAME}"
MODULES_PATH="/usr/share/icingaweb2/modules"
git clone ${REPO} "${MODULES_PATH}/${MODULE_NAME}" --branch "${MODULE_VERSION}"
icingacli module enable "${MODULE_NAME}"

MODULE_NAME=reactbundle
MODULE_VERSION=$(curl -s https://api.github.com/repos/Icinga/icingaweb2-module-reactbundle/releases/latest  | grep tag_name | cut -d '"' -f 4)
REPO="https://github.com/Icinga/icingaweb2-module-${MODULE_NAME}"
MODULES_PATH="/usr/share/icingaweb2/modules"
git clone ${REPO} "${MODULES_PATH}/${MODULE_NAME}" --branch "${MODULE_VERSION}"
icingacli module enable "${MODULE_NAME}"

ICINGAWEB_MODULEPATH="/usr/share/icingaweb2/modules"
REPO_URL="https://github.com/icinga/icingaweb2-module-director"
TARGET_DIR="${ICINGAWEB_MODULEPATH}/director"
MODULE_VERSION=$(curl -s https://api.github.com/repos/Icinga/icingaweb2-module-director/releases/latest  | grep tag_name | cut -d '"' -f 4)
git clone "${REPO_URL}" "${TARGET_DIR}" --branch ${MODULE_VERSION}
```


### Edit director configuration

```bash
cat <<EOF >> /etc/icinga2/conf.d/api-users.conf

object ApiUser "director" {
        password = "director"
        permissions = [ "*" ]
}
EOF
```

```bash
cat <<EOF >> /etc/icingaweb2/resources.ini

[director_db]
type = "db"
db = "pgsql"
host = "localhost"
port = "5432"
dbname = "director"
username = "director"
password = "director"
charset = "utf8"
use_ssl = "0"
EOF
```

```bash
mkdir /etc/icingaweb2/modules/director/
cat <<EOF >> /etc/icingaweb2/modules/director/config.ini
[db]
resource = "director_db"
EOF
```

```bash
cat <<EOF >> /etc/icingaweb2/modules/director/kickstart.ini
[config]
endpoint = icinga.mydomain.intra
host = 127.0.0.1
port = 5665
username = director
password = director
EOF
```

```bash
systemctl restart icinga2
icingacli module enable director
icingacli director kickstart run
icingacli director migration run --verbose
icingacli director migration pending --verbose
```

### install Background-Daemon

```bash
useradd -r -g icingaweb2 -d /var/lib/icingadirector -s /bin/false icingadirector
install -d -o icingadirector -g icingaweb2 -m 0750 /var/lib/icingadirector
```

```bash
MODULE_PATH=/usr/share/icingaweb2/modules/director
cp "${MODULE_PATH}/contrib/systemd/icinga-director.service" /etc/systemd/system/
systemctl daemon-reload

yum install php-posix -y

systemctl enable icinga-director.service --now
```

```bash
mkdir /etc/icingaweb2/modules/fileshipper
cat <<EOF >> /etc/icingaweb2/modules/fileshipper/imports.ini
[icinga2 - groups]
basedir = "/etc/icinga2/conf.d/groups/

[icinga2 - hosts]
basedir = "/etc/icinga2/conf.d/hosts/
EOF
```

