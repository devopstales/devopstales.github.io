---
title: Install Graylog3
url: https://devopstales.github.io/linux/graylog3-install/
date: 2019-06-24
keywords: Graylog, Pfsense, logging, CentOS
---


Graylog is defined in terms of log management platform for collecting, indexing, and analyzing both structured and unstructured data from almost any source.
<!--more-->

### Install requirement
```
yum install epel-release -y
yum install java-1.8.0-openjdk-headless.x86_64 pwgen nano -y
java -version
```

### Set Timezone
```
rm -f /etc/localtime
ln -s /usr/share/zoneinfo/CET /etc/localtime

yum install -y ntp
ntpd
```

### Elasticsearch
```
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

echo '[elasticsearch-6.x]
name=Elasticsearch repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
' | tee /etc/yum.repos.d/elasticsearch.repo

sudo yum -y install elasticsearch

sudo -E sed -i -e 's/#cluster.name: my-application/cluster.name: graylog/' \
/etc/elasticsearch/elasticsearch.yml

systemctl restart elasticsearch
systemctl enable elasticsearch

curl -XGET 'http://localhost:9200/_cluster/health?pretty=true'
```

### Mongodb

```
echo '[mongodb-org-4.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.0.asc' | tee /etc/yum.repos.d/mongodb-org.repo

yum -y install mongodb-org

systemctl restart mongod
systemctl enable  mongod
```

### Graylog3
```
rpm -Uvh https://packages.graylog2.org/repo/packages/graylog-3.3-repository_latest.rpm
yum -y install graylog-server

SECRET=$(pwgen -s 96 1)
sudo -E sed -i -e 's/password_secret =.*/password_secret = '$SECRET'/' /etc/graylog/server/server.conf
PASSWORD=$(echo -n Password1 | sha256sum | awk '{print $1}')
sudo -E sed -i -e 's/root_password_sha2 =.*/root_password_sha2 = '$PASSWORD'/' /etc/graylog/server/server.conf

# Set to your timezone
sudo -E sed -i -e 's/#root_timezone = UTC/root_timezone = CET/' /etc/graylog/server/server.conf

# Set to your email
sudo -E sed -i -e 's/#root_email = ""/root_email = "admin@devopstales.intra"/' /etc/graylog/server/server.conf
sudo -E sed -i -e 's/elasticsearch_shards = 4/elasticsearch_shards = 1/' /etc/graylog/server/server.conf
sudo -E sed -i -e 's/#http_bind_address = 127.0.0.1:9000/http_bind_address = 127.0.0.1:9400/' /etc/graylog/server/server.conf

# got ta https://dev.maxmind.com/geoip/geoip2/geolite2/ and download
# or use an old one
wget -t0 -c https://github.com/DocSpring/geolite2-city-mirror/raw/master/GeoLite2-City.tar.gz
tar -xvf GeoLite2-City.tar.gz
cp GeoLite2-City_*/GeoLite2-City.mmdb /etc/graylog/server

systemctl daemon-reload
systemctl restart graylog-server
systemctl enable graylog-server

tailf /var/log/graylog-server/server.log

If everything goes well, you should see below message in the logfile:
2019-06-20T13:37:04.059Z INFO  [ServerBootstrap] Graylog server up and running.
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


sudo yum install -y grafana
grafana-cli plugins install grafana-piechart-panel

sudo -E sed -i -e 's/;http_addr =/http_addr = 127.0.0.1/' /etc/grafana/grafana.ini

systemctl start grafana-server
systemctl status grafana-server
systemctl enable grafana-server
```

### Nginx Proxy
```
yum install nginx -y

echo 'server {
    listen 80;
    server_name graylog.devopstales.intra;

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
    server_name grafana.devopstales.intra;

    location / {
      proxy_set_header Host $http_host;
      proxy_set_header X-Forwarded-Host $host;
      proxy_set_header X-Forwarded-Server $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_pass       http://127.0.0.1:3000;
    }
}' > /etc/nginx/conf.d/grafana.conf

nginx -t
systemctl restart nginx
systemctl enable nginx
```

