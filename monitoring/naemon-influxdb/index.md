---
title: Install Nemon with Influxdb storage
url: https://devopstales.github.io/monitoring/naemon-influxdb/
date: 2019-01-01
keywords: nagios, naemon, nosql, influxdb, grafana
---


I prefer to use Naemon (a fork of nagos) with Influxdb as a storage for graphical data.

<!--more-->

### Install Naemon
```
yum install epel-release nano -y
yum install httpd php php-gd -y

rpm -Uvh "https://labs.consol.de/repo/stable/rhel7/x86_64/labs-consol-stable.rhel7.noarch.rpm"

yum install naemon* -y
yum install nagios-plugins nagios-plugins-all nagios-plugins-nrpe nrpe -y

nano /etc/php.ini
date.timezone = Europe/Budapest


systemctl enable httpd
systemctl enable naemon
systemctl start httpd
systemctl start naemon

htpasswd /etc/thruk/htpasswd thrukadmin
# http://SERVER-IP/naemon
```

### Install Infludxb
```
cat <<EOF | sudo tee /etc/yum.repos.d/influxdb.repo
[influxdb]
name = InfluxDB Repository - RHEL \$releasever
baseurl = https://repos.influxdata.com/rhel/\$releasever/\$basearch/stable
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdb.key
EOF

yum install influxdb -y

nano /etc/influxdb/influxdb.conf
[http]
   enabled = true
   bind-address = "localhost:8086"
   auth-enabled = false

systemctl start influxdb
systemctl enable influxdb
```

### Configurate Naemon
```
sed -i "s@^process_performance_data=0@#process_performance_data=0@" /etc/naemon/naemon.cfg
# config nagios
nano /etc/naemon/module-conf.d/nagios_nagflux.cfg
process_performance_data=1

host_perfdata_file=/var/naemon/host-perfdata
host_perfdata_file_template=DATATYPE::HOSTPERFDATA\tTIMET::$TIMET$\tHOSTNAME::$HOSTNAME$\tHOSTPERFDATA::$HOSTPERFDATA$\tHOSTCHECKCOMMAND::$HOSTCHECKCOMMAND$
host_perfdata_file_mode=a
host_perfdata_file_processing_interval=15
host_perfdata_file_processing_command=process-host-perfdata-file-nagflux

service_perfdata_file=/var/naemon/service-perfdata
service_perfdata_file_template=DATATYPE::SERVICEPERFDATA\tTIMET::$TIMET$\tHOSTNAME::$HOSTNAME$\tSERVICEDESC::$SERVICEDESC$\tSERVICEPERFDATA::$SERVICEPERFDATA$\tSERVICECHECKCOMMAND::$SERVICECHECKCOMMAND$
service_perfdata_file_mode=a
service_perfdata_file_processing_interval=15
service_perfdata_file_processing_command=process-service-perfdata-file-nagflux

chown naemon:naemon /etc/naemon/module-conf.d/nagios_nagflux.cfg

nano /etc/naemon/conf.d/histou.cfg
define command {
    command_name    process-host-perfdata-file-nagflux
    command_line    /bin/mv /var/naemon/host-perfdata /var/nagflux/perfdata/$TIMET$.perfdata.host
    }

define command {
    command_name    process-service-perfdata-file-nagflux
    command_line    /bin/mv /var/naemon/service-perfdata /var/nagflux/perfdata/$TIMET$.perfdata.service
    }

define host {
   name       host-grafana
   action_url http://192.168.10.112/grafana/dashboard/script/histou.js?host=$HOSTNAME$&theme=light&annotations=true
   notes_url   http://192.168.10.112/dokuwiki/doku.php?id=inventory:$HOSTNAME$
   register   0
}

define service {
   name       service-grafana
   action_url http://192.168.10.112/grafana/dashboard/script/histou.js?host=$HOSTNAME$&service=$SERVICEDESC$&theme=light&annotations=true
   register   0
}

mkdir /var/naemon/
chown -R naemon:naemon /var/naemon/

cd /etc/thruk/ssi/
cp extinfo-header.ssi.example extinfo-header.ssi
cp status-header.ssi.example status-header.ssi

systemctl restart naemon

ll /var/naemon/
ll /var/nagflux/perfdata/
```

### Install nagflux
```
cd /usr/bin/
wget https://github.com/Griesbacher/nagflux/releases/download/v0.4.1/nagflux
chmod +x nagflux

mkdir -p /var/nagflux/perfdata
mkdir -p /var/nagflux/spool
chown -R naemon:apache /var/nagflux

mkdir /etc/nagflux
cat <<EOF | sudo tee /etc/nagflux/config.gcfg
[main]
NagiosSpoolfileFolder = "/var/nagflux/perfdata"
NagiosSpoolfileWorker = 1
InfluxWorker = 2
MaxInfluxWorker = 5
DumpFile = "/var/log/nagflux/nagflux.dump"
NagfluxSpoolfileFolder = "/var/nagflux/spool"
FieldSeparator = "&"
BufferSize = 1000
FileBufferSize = 65536
DefaultTarget = "Influxdb"

[Log]
LogFile = "/var/log/nagflux/nagflux.log"
MinSeverity = "INFO"

[InfluxDBGlobal]
CreateDatabaseIfNotExists = true
NastyString = ""
NastyStringToReplace = ""
HostcheckAlias = "hostcheck"

[InfluxDB "nagflux"]
Enabled = true
Version = 1.0
Address = "http://localhost:8086"
Arguments = "precision=ms&db=nagflux&u=admin&p=Password1"
StopPullingDataIfDown = true

[Livestatus]
#tcp or file
Type = "file"
#tcp: 127.0.0.1:6557 or file /var/run/live
Address = "/var/cache/naemon/live"
MinutesToWait = 3
Version = ""
EOF

mkdir /var/log/nagflux
mkdir /var/nagflux

cat <<EOF | sudo tee /etc/systemd/system/nagflux.service
[Unit]
Description=A connector which transforms performancedata from Nagios/Icinga(2)/Naemon to InfluxDB/Elasticsearch
Documentation=https://github.com/Griesbacher/nagflux
After=network-online.target

[Service]
User=root
Group=root
ExecStart=/usr/bin/nagflux -configPath /etc/nagflux/config.gcfg
Restart=on-failure

[Install]
WantedBy=multi-user.target
Alias=nagflux.service
EOF

systemctl daemon-reload
systemctl start nagflux
systemctl enable nagflux

tailf /var/log/nagflux/nagflux.log
```

### Install grafana
```
curl -s https://packagecloud.io/install/repositories/grafana/stable/script.rpm.sh | sudo bash
yum install grafana -y

cp /etc/grafana/grafana.ini /etc/grafana/grafana.ini.bak
echo "" > /etc/grafana/grafana.ini
nano /etc/grafana/grafana.ini
[paths]
logs = /var/log/grafana

[log]
mode = file
[log.file]
level =  Info
daily_rotate = true

[server]
http_port = 3000
http_addr = 0.0.0.0
domain = localhost
root_url = %(protocol)s://%(domain)s/grafana/
enable_gzip = false

[snapshots]
external_enabled = false

[security]
disable_gravatar = true
# same username and password for thruk
admin_user = thrukadmin
admin_password = Password1

[users]
allow_sign_up = false
default_theme = light

[auth.basic]
enabled = false

[auth.proxy]
enabled = true
auto_sign_up = true

[alerting]
enabled = true
execute_alerts = true

nano /etc/httpd/conf.d/grafana.conf
<IfModule !mod_proxy.c>
    LoadModule proxy_module /usr/lib64/httpd/modules/mod_proxy.so
</IfModule>
<IfModule !mod_proxy_http.c>
    LoadModule proxy_http_module /usr/lib64/httpd/modules/mod_proxy_http.so
</IfModule>

<Location /grafana>
    ProxyPass http://127.0.0.1:3000 retry=0 disablereuse=On
    ProxyPassReverse http://127.0.0.1:3000/grafana
    RewriteEngine On
    RewriteRule .* - [E=PROXY_USER:%{LA-U:REMOTE_USER},NS]
    SetEnvIf Request_Protocol ^HTTPS.* IS_HTTPS=1
    SetEnvIf Authorization "^.+$" IS_BASIC_AUTH=1
    # without thruk cookie auth, use the proxy user from the rewrite rule above
    RequestHeader set X-WEBAUTH-USER "%{PROXY_USER}s"  env=IS_HTTPS
    RequestHeader set X-WEBAUTH-USER "%{PROXY_USER}e"  env=!IS_HTTPS
    # when thruk cookie auth is used, fallback to remote user directly
    RequestHeader set X-WEBAUTH-USER "%{REMOTE_USER}e" env=!IS_BASIC_AUTH
    RequestHeader unset Authorization
</Location>

echo "
apiVersion: 1

deleteDatasources:
  - name: nagflux

datasources:
- name: nagflux
  type: influxdb
  url: http://localhost:8086
  access: proxy
  database: nagflux
  isDefault: true
  version: 1
  editable: true
" > /etc/grafana/provisioning/datasources/nagflux.yaml

systemctl start grafana-server
systemctl enable grafana-server
systemctl restart httpd


# http://SERVER-IP:3000
# admin/admin

# datasource:
nagflux
influxdb
http://localhost:8086
```

### Install histou
```
cd /tmp
wget -O histou.tar.gz https://github.com/Griesbacher/histou/archive/v0.4.3.tar.gz
mkdir -p /var/www/html/histou
cd /var/www/html/histou
tar xzf /tmp/histou.tar.gz --strip-components 1
cp histou.ini.example histou.ini
cp histou.js /usr/share/grafana/public/dashboards/

nano /usr/share/grafana/public/dashboards/histou.js
var url = 'http://192.168.10.112/histou/';

systemctl restart httpd
systemctl restart grafana-server

# http://192.168.10.112/histou/?host=localhost&service=PING
# http://192.168.10.112:3000/dashboard/script/histou.js?host=localhost&service=PING

# nagios config

sed -i '/name.*generic-host/a\        use                             host-grafana' /etc/naemon/conf.d/templates/hosts.cfg
sed -i '/name.*generic-service/a\        use                             service-grafana' /etc/naemon/conf.d/templates/services.cfg

systemctl restart naemon
```

### Inpluxdb commands
```
influx
create database nagflux;
CREATE USER "admin" WITH PASSWORD 'Password1' WITH ALL PRIVILEGES;
show DATABASES;
USE nagflux;
select * from /.*/ limit 1;
```

