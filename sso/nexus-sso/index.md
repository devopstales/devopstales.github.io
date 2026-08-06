---
title: Sonatype Nexus SSO
url: https://devopstales.github.io/sso/nexus-sso/
date: 2019-05-23
keywords: Keycloak, oauth2, SSO, OIDC, nexus
---


Nexus Repository OSS is an artifact repository with universal support for popular formats.
<!--more-->

### Install Nexus
```
cd /opt
wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz
tar xvf latest-unix.tar.gz -C /opt
ln -s /opt/nexus-3.16.1-02/ /opt/nexus

adduser -s /bin/false nexus
chown -R nexus:nexus /opt/nexus
chown -R nexus:nexus /opt/sonatype-work/

echo 'run_as_user="nexus"' > /opt/nexus/bin/nexus.rc

nano /opt/nexus/bin/nexus
INSTALL4J_JAVA_HOME_OVERRIDE=/usr/lib/jvm/jre-1.8.0
```

### Create sistemd serice for Nexus
```
echo '[Unit]
Description=nexus service
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
User=nexus
Restart=on-abort

[Install]
WantedBy=multi-user.target' > /etc/systemd/system/nexus.service
```

### Start Nexus
```
sudo systemctl daemon-reload
sudo systemctl enable nexus.service
sudo systemctl start nexus.service

tailf /opt/sonatype-work/nexus3/log/nexus.log
### To check, point your browser to http://localhost:8081. Default username is admin with password admin123.
```

### Install Keycloak authentication plugin
```
NEXUS_PLUGINS=/opt/nexus/system
KEYCLOAK_PLUGIN_VERSION=0.3.3-SNAPSHOT
cd /opt
mkdir -p ${NEXUS_PLUGINS}/org/github/flytreeleft/nexus3-keycloak-plugin/${KEYCLOAK_PLUGIN_VERSION}/
cd ${NEXUS_PLUGINS}/org/github/flytreeleft/nexus3-keycloak-plugin/${KEYCLOAK_PLUGIN_VERSION}/
wget https://github.com/flytreeleft/nexus3-keycloak-plugin/releases/download/${KEYCLOAK_PLUGIN_VERSION}/nexus3-keycloak-plugin-${KEYCLOAK_PLUGIN_VERSION}.jar
chmod 644 ${NEXUS_PLUGINS}/org/github/flytreeleft/nexus3-keycloak-plugin/${KEYCLOAK_PLUGIN_VERSION}/nexus3-keycloak-plugin-${KEYCLOAK_PLUGIN_VERSION}.jar
echo "mvn\\:org.github.flytreeleft/nexus3-keycloak-plugin/${KEYCLOAK_PLUGIN_VERSION} = 200" >> /opt/nexus/etc/karaf/startup.properties
```
Login to your Keycloak, and navigate relm > client
![Example image](/img/include/nexus_sso1.webp)<br><br>
Configurate Service Account Roles
![Example image](/img/include/nexus_sso2.webp)<br><br>
Configurate User Roles
![Example image](/img/include/nexus_sso4.webp)<br><br>

```
nano /opt/nexus/etc/keycloak.json
{
  "realm": "mydomain",
  "auth-server-url": "http://nexus.devopstales.intra:8080/auth",
  "ssl-required": "external",
  "resource": "web",
  "credentials": {
    "secret": "41e39b6b-e23a-4fb1-be21-d30c02941ffc"
  },
  "confidential-port": 0
}

systemct restart nexus
```
After login to nexus you can navigate to the realm administration. Activate the Keycloak Authentication Realm plugin by dragging it to the right hand side.
![Example image](/img/include/nexus_sso3.webp)<br><br>
Mapp the Keycloak roles to nexus
![Example image](/img/include/nexus_sso5.webp)<br><br>
Go to server administration > system > capabilities > add <br>
type: Ruth auth <br>
HTTP Header: X-Proxy-REMOTE-USER <br>
![Example image](/img/include/nexus_sso6.webp)<br><br>

```
yum install mod_auth_openidc httpd mod_ssl -y
nano /etc/httpd/conf.d/nexus-site.conf
ProxyRequests Off
ProxyPreserveHost On

<VirtualHost *:80>
    ServerName nexus.devopstales.intra
     Redirect permanent / https://nexus.devopstales.intra
    ErrorLog /var/log/httpd/error.log
    CustomLog /var/log/httpd/access.log common
</VirtualHost>

<VirtualHost *:443>
    ServerAdmin webmaster@example.com
    ServerName nexus.devopstales.intra
    ServerAlias www.nexus.devopstales.intra
    DirectoryIndex index.html index.php

    SSLEngine on
    SSLCertificateFile /etc/httpd/ssl/domain.pem
    SSLCertificateKeyFile /etc/httpd/ssl/domain.pem
    SSLCertificateChainFile /etc/httpd/ssl/domain.pem

    AllowEncodedSlashes NoDecode
    AllowEncodedSlashes On
    RequestHeader set X-Forwarded-Proto "https"

    # keycloak
    OIDCProviderMetadataURL https://nexus.devopstales.intra:8443/auth/realms/mydomain/.well-known/openid-configuration
    OIDCSSLValidateServer Off
    OIDCClientID web
    OIDCClientSecret 41e39b6b-e23a-4fb1-be21-d30c02941ffc
    OIDCRedirectURI https://nexus.devopstales.intra/redirect_uri
    OIDCCryptoPassphrase passphrase
    OIDCJWKSRefreshInterval 3600
    OIDCScope "openid email profile"
    # maps the prefered_username claim to the REMOTE_USER environment variable
    OIDCRemoteUserClaim preferred_username

    <Location />
      AuthType openid-connect
      Require valid-user
      RequestHeader set "X-Proxy-REMOTE-USER" %{REMOTE_USER}s
      ProxyPass http://localhost:8081/ nocanon
      ProxyPassReverse http://localhost:8081/
    </Location>

    ErrorLog /var/log/httpd/error.log
    CustomLog /var/log/httpd/access.log common
</VirtualHost>

# secure neus server
nano /opt/nexus/etc/nexus-default.properties
application-host=127.0.0.1
```

