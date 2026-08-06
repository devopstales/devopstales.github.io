---
title: Install icinga director modules to Icingaweb2
url: https://devopstales.github.io/monitoring/icinga2_director/
date: 2019-12-13
keywords: icinga, icinga2, nagios
---


In this tutorial I will show you how to install Icingaweb2 module director.

<!--more-->
Icinga irector is designed for those who want to automate their configuration deployment and those who want to grant easy access for there users to the Icinga2 configuration.

### Install dependency
```
yum install git -y
yum install rh-php71-php-curl rh-php71-php-pcntl rh-php71-php-posix rh-php71-php-sockets rh-php71-php-xml rh-php71-php-zip -y
```

```
nano
local   director      director                        md5
host    director      director      127.0.0.1/32      md5
host    director      director      ::1/128           md5

systemctl restart postgresql-10
```

### Install Icingaweb2 modules
```
MODULE_NAME=ipl
MODULE_VERSION=v0.4.0
REPO="https://github.com/Icinga/icingaweb2-module-${MODULE_NAME}"
MODULES_PATH="/usr/share/icingaweb2/modules"
git clone ${REPO} "${MODULES_PATH}/${MODULE_NAME}" --branch "${MODULE_VERSION}"
icingacli module enable "${MODULE_NAME}"

MODULE_NAME=incubator
MODULE_VERSION=v0.5.0
REPO="https://github.com/Icinga/icingaweb2-module-${MODULE_NAME}"
MODULES_PATH="/usr/share/icingaweb2/modules"
git clone ${REPO} "${MODULES_PATH}/${MODULE_NAME}" --branch "${MODULE_VERSION}"
icingacli module enable "${MODULE_NAME}"

MODULE_NAME=reactbundle
MODULE_VERSION=v0.7.0
REPO="https://github.com/Icinga/icingaweb2-module-${MODULE_NAME}"
MODULES_PATH="/usr/share/icingaweb2/modules"
git clone ${REPO} "${MODULES_PATH}/${MODULE_NAME}" --branch "${MODULE_VERSION}"
icingacli module enable "${MODULE_NAME}"

ICINGAWEB_MODULEPATH="/usr/share/icingaweb2/modules"
REPO_URL="https://github.com/icinga/icingaweb2-module-director"
TARGET_DIR="${ICINGAWEB_MODULEPATH}/director"
MODULE_VERSION="1.7.2"
git clone "${REPO_URL}" "${TARGET_DIR}"
cd "${TARGET_DIR}"
git fetch && git fetch --tags
git checkout "{MODULE_VERSION}"
restorecon -R "${TARGET_DIR}"

MODULE_NAME=fileshipper
MODULE_VERSION=v1.1.0
REPO="https://github.com/Icinga/icingaweb2-module-${MODULE_NAME}"
MODULES_PATH="/usr/share/icingaweb2/modules"
git clone ${REPO} "${MODULES_PATH}/${MODULE_NAME}" --branch "${MODULE_VERSION}"
icingacli module enable "${MODULE_NAME}"
```

### Create db for director
```
sudo -u postgres psql -c "CREATE DATABASE director WITH ENCODING 'utf8';"
sudo -u postgres psql director -q -c "CREATE USER director WITH PASSWORD 'director';
GRANT ALL PRIVILEGES ON DATABASE director TO director;
CREATE EXTENSION pgcrypto;"
sudo -u postgres psql director < /usr/share/icingaweb2/modules/director/schema/pgsql.sql
```

### Edit director configuration
```
cat <<EOF >> /etc/icinga2/zones.conf

object Zone "director-global" {
  global = true
}
EOF

cat <<EOF >> /etc/icinga2/conf.d/api-users.conf

object ApiUser "director" {
        password = "director"
        permissions = [ "*" ]
}
EOF

cat <<EOF >> /etc/icingaweb2/resources.ini

[icinga_director]
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

mkdir /etc/icingaweb2/modules/director/
cat <<EOF >> /etc/icingaweb2/modules/director/config.ini
[db]
resource = "icinga_director"
EOF

cat <<EOF >> /etc/icingaweb2/modules/director/kickstart.ini
[config]
endpoint = icinga.devopstales.intra
host = 127.0.0.1
port = 5665
username = director
password = director
EOF
```

```
icingacli module enable director
icingacli director kickstart run
icingacli director migration run --verbose
icingacli director migration pending --verbose
```

```
mkdir /etc/icingaweb2/modules/fileshipper
cat <<EOF >> /etc/icingaweb2/modules/fileshipper/imports.ini
[icinga2 - groups]
basedir = "/etc/icinga2/conf.d/groups/

[icinga2 - hosts]
basedir = "/etc/icinga2/conf.d/hosts/
EOF
```

