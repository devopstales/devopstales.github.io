---
title: Install Graylog5
url: https://devopstales.github.io/linux/graylog5-install/
date: 2022-12-19
keywords: Graylog, Pfsense, logging, CentOS
---



Graylog is defined in terms of log management platform for collecting, indexing, and analyzing both structured and unstructured data from almost any source.
<!--more-->

### Install requirement
```bash
yum install epel-release -y
yum install java-17-openjdk-headless.x86_64 pwgen nano wget curl git -y
java -version
```

```bash
setenforce 0
sed -i 's/=\(enforcing\|permissive\)/=disabled/g' /etc/sysconfig/selinux
sed -i 's/=\(enforcing\|permissive\)/=disabled/g' /etc/selinux/config
```

>> Important to configure the time correctly for the graphs to populating correctly

### Set Timezone
```bash
dnf install -y chrony ntpstat

timedatectl set-timezone CET
timedatectl set-ntp true
systemctl enable chronyd --now
```

### OpenSearch 2.x
```bash
rpm --import https://artifacts.opensearch.org/publickeys/opensearch.pgp

curl -SL https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/opensearch-2.x.repo \
-o /etc/yum.repos.d/opensearch-2.x.repo

yum install opensearch -y
```

Configure the OpenSearch

```bash
swapoff -a

echo "* hardnofile 65535" >> /etc/security/limits.conf
echo "* soft nofile 65535" >> /etc/security/limits.conf

echo "vm.max_map_count=262144" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.conf
sysctl -p
cat /proc/sys/vm/max_map_count

sed -i "s|::1|#::1|" /etc/hosts


nano /etc/opensearch/opensearch.yml
cluster.name: graylog
...
network.host: 127.0.0.1
...
plugins.security.ssl.http.enabled: false
...
node.max_local_storage_nodes: "1"
...
discovery.type: single-node
action.auto_create_index: ".watches,.triggered_watches,.watcher-history-*"
bootstrap.memory_lock: true
```

You may prefer to disable transparent hugepages to improve performance before installing. 

```bash
cat > /etc/systemd/system/disable-transparent-huge-pages.service <<EOF
Description=Disable Transparent Huge Pages (THP)
DefaultDependencies=no
After=sysinit.target local-fs.target
[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never | tee /sys/kernel/mm/transparent_hugepage/enabled > /dev/null'
[Install]
WantedBy=basic.target
EOF

systemctl daemon-reload
systemctl enable disable-transparent-huge-pages.service
systemctl start disable-transparent-huge-pages.service
```

Edit service to disable memory lock

```bash
nano /usr/lib/systemd/system/opensearch.service

[Service]
LimitMEMLOCK=infinity
```

Add half of the host memory to the opensearch

```bash
nano /etc/opensearch/jvm.options

-Xms4g
-Xmx4g
```

Start end test OpenSearch

```bash
systemctl daemon-reload
systemctl restart opensearch
systemctl enable opensearch
systemctl status opensearch

curl -XGET 'http://admin:admin@localhost:9200/_cluster/health?pretty=true'
```

### Mongodb

```bash
echo '[mongodb-org-5.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/5.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-5.0.asc' | tee /etc/yum.repos.d/mongodb-org.repo

yum -y install mongodb-org

systemctl restart mongod
systemctl enable  mongod
systemctl status  mongod
```

### Graylog5
```bash
rpm -Uvh https://packages.graylog2.org/repo/packages/graylog-5.0-repository_latest.rpm
yum -y install graylog-server
```

>> Important to configure the time correctly for the graphs to populating correctly

Configure Graylog server

```bash
SECRET=$(pwgen -s 96 1)
sed -i -e 's/password_secret =.*/password_secret = '$SECRET'/' /etc/graylog/server/server.conf
PASSWORD=$(echo -n Password1 | sha256sum | awk '{print $1}')
sed -i -e 's/root_password_sha2 =.*/root_password_sha2 = '$PASSWORD'/' /etc/graylog/server/server.conf

# Set to your timezone
sed -i -e 's/#root_timezone = UTC/root_timezone = CET/' /etc/graylog/server/server.conf

# Set to your email
sed -i -e 's/#root_email = ""/root_email = "admin@devopstales.intra"/' /etc/graylog/server/server.conf
sed -i -e 's/elasticsearch_shards = 4/elasticsearch_shards = 1/' /etc/graylog/server/server.conf
sed -i -e 's/#http_bind_address = 127.0.0.1:9000/http_bind_address = 127.0.0.1:9400/' /etc/graylog/server/server.conf
sed -i -e "s|#elasticsearch_hosts = http://node1:9200,http://user:password@node2:19200|elasticsearch_hosts = http://admin:admin@127.0.0.1:9200|" /etc/graylog/server/server.conf

# go to https://dev.maxmind.com/geoip/geoip2/geolite2/ and download
# or use an old one
cd /etc/graylog/server
wget https://github.com/socfortress/Wazuh-Rules/releases/download/1.0/GeoLite2-City.mmdb
wget https://github.com/socfortress/Wazuh-Rules/releases/download/1.0/GeoLite2-ASN.mmdb

systemctl daemon-reload
systemctl restart graylog-server
systemctl enable graylog-server

tail -f /var/log/graylog-server/server.log

If everything goes well, you should see below message in the logfile:
2022-12-19T13:37:04.059Z INFO  [ServerBootstrap] Graylog server up and running.
```

### Install Grafana
```
echo '[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
' > /etc/yum.repos.d/grafana.repo


yum install -y grafana
grafana-cli plugins install grafana-piechart-panel
grafana-cli plugins install netsage-sankey-panel
grafana-cli plugins install grafana-worldmap-panel
grafana-cli plugins install savantly-heatmap-panel

sed -i -e 's/;http_addr =/http_addr = 127.0.0.1/' /etc/grafana/grafana.ini

systemctl start grafana-server
systemctl status grafana-server
systemctl enable grafana-server
```

### OpenSearch Dashboard
```
curl -SL https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/2.x/opensearch-dashboards-2.x.repo \
-o /etc/yum.repos.d/opensearch-dashboards-2.x.repo


yum install opensearch-dashboards -y

nano /etc/opensearch-dashboards/opensearch_dashboards.yml
opensearch.hosts: [http://localhost:9200]

systemctl restart opensearch-dashboards
systemctl enable opensearch-dashboards
systemctl status opensearch-dashboards
```

### Nginx Proxy
```
yum install nginx -y

echo 'server {
    listen 80;
    server_name graylog.mydomain.intra;

    location / {
      proxy_set_header Host $http_host;
      proxy_set_header X-Forwarded-Host $host;
      proxy_set_header X-Forwarded-Server $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Graylog-Server-URL http://$server_name/;
      proxy_pass       http://127.0.0.1:9400;
    }
}' > /etc/nginx/conf.d/graylog.conf

echo 'server {
    listen 80;
    server_name grafana.mydomain.intra;

    location / {
      proxy_set_header Host $http_host;
      proxy_set_header X-Forwarded-Host $host;
      proxy_set_header X-Forwarded-Server $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_pass       http://127.0.0.1:3000;
    }
}' > /etc/nginx/conf.d/grafana.conf

echo 'server {
    listen 80;
    server_name kibana.mydomain.intra;

    location / {
      proxy_set_header Host $http_host;
      proxy_set_header X-Forwarded-Host $host;
      proxy_set_header X-Forwarded-Server $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_pass       http://127.0.0.1:5601;
    }
}' > /etc/nginx/conf.d/kibana.conf

nginx -t
systemctl restart nginx
systemctl enable nginx
```

