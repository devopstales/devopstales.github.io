---
title: Install keycloak with postgresql
url: https://devopstales.github.io/sso/keycloak2/
date: 2019-04-05
keywords: Keycloak, oauth2, SSO, OIDC
---


Keycloak is an open source identity and access management solution.

<!--more-->

### Install dependencies
```
yum install -y epel-release
yum install -y java-1.8.0-openjdk-headless tmux nano mariadb-server unzip httpd

cd /opt
# https://jdbc.postgresql.org/download.html
wget https://jdbc.postgresql.org/download/postgresql-42.2.5.jar
```

### Install and configure database
In a previous post I wrote about how to [Install PostgreSQL 10]({{< relref "/linux/install-postgresql.md#install-postgresql-10-on-centos-7" >}})

```
nano /var/lib/pgsql/10/data/pg_hba.conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             keycloak                                md5

su - postgres
createuser keycloak
psql
ALTER USER keycloak WITH ENCRYPTED password 'Password1';
CREATE DATABASE keycloak WITH ENCODING='UTF8' OWNER=keycloak;
\q
```

### Install keycloak
```
groupadd -r keycloak
useradd -m -d /var/lib/keycloak -s /sbin/nologin -r -g keycloak keycloak

mkdir -p /opt/keycloak/
cd /opt/keycloak/

# https://www.keycloak.org/downloads.html
wget https://downloads.jboss.org/keycloak/4.8.2.Final/keycloak-4.8.2.Final.tar.gz

tar -xzf keycloak-4.8.2.Final.tar.gz
ln -s /opt/keycloak/keycloak-4.8.2.Final /opt/keycloak/current
chown keycloak: -R /opt/keycloak
sudo -u keycloak chmod 700 /opt/keycloak/current/standalone

mkdir /var/log/keycloak
chown keycloak: -R /var/log/keycloak

chown keycloak: -R /opt/keycloak
sudo -u keycloak chmod 700 /opt/keycloak/current/standalone
```

```
echo '[Unit]
Description=Keycloak
After=network.target syslog.target

[Service]
Type=idle
User=keycloak
Group=keycloak
ExecStart=/opt/keycloak/current/bin/standalone.sh -b 0.0.0.0
TimeoutStartSec=600
TimeoutStopSec=600

StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=keycloak

[Install]
WantedBy=multi-user.target
' > /etc/systemd/system/keycloak.service
```

```
echo 'if $programname == "keycloak" then /var/log/keycloak/jboss.log
& stop
'>/etc/rsyslog.d/keycloak.conf

systemctl daemon-reload
service rsyslog restart
systemctl start keycloak.service
```

### Configure wildfly
```
cd /opt/keycloak/current/modules
mkdir -p org/postgresql/main
cp /opt/postgresql-42.2.5.jar .

echo '<?xml version="1.0" ?>
<module xmlns="urn:jboss:module:1.3" name="org.postgresql">

    <resources>
        <resource-root path="postgresql-42.2.5.jar"/>
	</resources>

	<dependencies>
		<module name="javax.api"/>
		<module name="javax.transaction.api"/>
	</dependencies>
</module>' > org/postgresql/main/module.xml
```

```
cd /opt/keycloak/current/standalone/configuration/
nano standalone.xml
...
        <datasources>
				<datasource jndi-name="java:jboss/datasources/KeycloakDS" pool-name="KeycloakDS" enabled="true" use-java-context="true">
					<connection-url>jdbc:postgresql://localhost:5432/keycloak</connection-url>
					<driver>postgresql</driver>
					<pool>
						<max-pool-size>20</max-pool-size>
					</pool>
					<security>
						<user-name>keycloak</user-name>
						<password>Password1</password>
					</security>
				</datasource>
...
        <drivers>
					<driver name="postgresql" module="org.postgresql">
						<xa-datasource-class>org.postgresql.xa.PGXADataSource</xa-datasource-class>
				</driver>
...
<default-bindings context-service="java:jboss/ee/concurrency/context/default" datasource="java:jboss/datasources/KeycloakDS"
```

```
cd /opt/keycloak/current
./bin/add-user-keycloak.sh -u admin -p Password1 -r master
systemctl restart keycloak.service
```

### Configurate proxy
```
echo '<VirtualHost *:80>
    ServerName sso.devopstales.intra

    ProxyPreserveHost On
#    SSLProxyEngine On
#    SSLProxyCheckPeerCN on
#    SSLProxyCheckPeerExpire on
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Port "80" #443
    ProxyPass / http://127.0.0.1:8080/
    ProxyPassReverse / http://127.0.0.1:8080/
</VirtualHost>' > /etc/apache2/sites-available/keycloak.conf

sudo a2enmod headers
a2enmod proxy
a2enmod rewrite
a2ensite keycloak.confcd
service httpd restart

# go to sso.devopstales.intra
# login admin / Password1
```

