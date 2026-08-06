---
title: Install keycloak with mysql
url: https://devopstales.github.io/sso/keycloak1/
date: 2019-04-05
keywords: Keycloak, oauth2, SSO, OIDC
---


Keycloak is an open source identity and access management solution.

<!--more-->

### Install dependencies
```
yum install -y epel-release
yum install -y java-1.8.0-openjdk-headless tmux nano mariadb-server unzip nginx

cd /opt/
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.47.zip
unzip mysql-connector-java-5.1.47.zip
```

### Configure database
```
service mariadb start

mysql -uroot
CREATE DATABASE keycloak CHARACTER SET utf8 COLLATE utf8_unicode_ci;
GRANT ALL PRIVILEGES ON keycloak.* TO 'keycloak'@'%' identified by 'Password1';
GRANT ALL PRIVILEGES ON keycloak.* TO 'keycloak'@'localhost' identified by 'Password1';
FLUSH privileges;
exit;
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
cd /opt/keycloak/current/

./bin/jboss-cli.sh -c 'module add --name=org.mysql  --dependencies=javax.api,javax.transaction.api --resources=/opt/mysql-connector-java-5.1.47/mysql-connector-java-5.1.47.jar'

./bin/jboss-cli.sh 'embed-server,/subsystem=datasources/jdbc-driver=org.mysql:add(driver-name=org.mysql,driver-module-name=org.mysql,driver-class-name=com.mysql.jdbc.Driver)'

./bin/jboss-cli.sh 'embed-server,/subsystem=datasources/data-source=KeycloakDS:remove'

./bin/jboss-cli.sh 'embed-server,/subsystem=datasources/data-source=KeycloakDS:add(driver-name=org.mysql,enabled=true,use-java-context=true,connection-url="jdbc:mysql://localhost:3306/keycloak?useSSL=false&amp;useLegacyDatetimeCode=false&amp;serverTimezone=Europe/Budapest&amp;characterEncoding=UTF-8",jndi-name="java:/jboss/datasources/KeycloakDS",user-name=keycloak,password="Password1",valid-connection-checker-class-name=org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLValidConnectionChecker,validate-on-match=true,exception-sorter-class-name=org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLValidConnectionChecker)'

./bin/add-user-keycloak.sh -u admin -p Password1 -r master

# for nginx proxy
./bin/jboss-cli.sh 'embed-server,/subsystem=undertow/server=default-server/http-listener=default:write-attribute(name=proxy-address-forwarding,value=true)'

./bin/jboss-cli.sh 'embed-server,/socket-binding-group=standard-sockets/socket-binding=proxy-https:add(port=443)'

./bin/jboss-cli.sh 'embed-server,/subsystem=undertow/server=default-server/http-listener=default:write-attribute(name=redirect-socket,value=proxy-https)'

# disabla color in log
./bin/jboss-cli.sh -c '/subsystem=logging/console-handler=CONSOLE:write-attribute(name=named-formatter, value=PATTERN)'
```


### Configurate proxy
```
systemctl restart keycloak.service

echo 'upstream keycloak {
    # Use IP Hash for session persistence
    ip_hash;

    # List of Keycloak servers
    server 127.0.0.1:8080;
}


server {
    listen 80;
    server_name sso.devopstales.intra;

    # Redirect all HTTP to HTTPS
    location / {
      return 301 https://$server_name$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name sso.devopstales.intra;

    ssl_certificate /etc/nginx/ssl/domain.pem;
    ssl_certificate_key /etc/nginx/ssl/domain.pem;
    ssl_session_cache shared:SSL:1m;
    ssl_prefer_server_ciphers on;

    location / {
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto  $scheme;
      proxy_pass http://keycloak;
    }
}
' > /etc/nginx/conf.d/keycloak.conf

mkdir /etc/nginx/ssl

systemctl restart nginx

# go to sso.devopstales.intra
# login admin / Password1
```

