---
title: Install Prometheus with Influxdb storage
url: https://devopstales.github.io/monitoring/prometheus-influxdb/
date: 2019-02-02
keywords: promethsus, noSQL, influxdb
---


Use Influxdb to as storage for Prometheus.

<!--more-->

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
```

### Configure Influxdb
```
nano /etc/influxdb/influxdb.conf
[http]
   enabled = true
   bind-address = "localhost:8086"
   auth-enabled = false
```

### Configure Prometheus
```
nano /etc/prometheus/prometheus.yml
remote_write:
  - url: "http://localhost:8086/api/v1/prom/write?db=prometheus"

remote_read:
  - url: "http://localhost:8086/api/v1/prom/read?db=prometheus"

# with authentication
#remote_write:
#  - url: "http://localhost:8086/api/v1/prom/write?db=prometheus&u=username&p=password"

#remote_read:
#  - url: "http://localhost:8086/api/v1/prom/read?db=prometheus&u=username&p=password"
```


```
systemctl start influxdb
systemctl enable influxdb

echo 'CREATE DATABASE "prometheus"' | influx

systemctl start prometheus
systemctl status prometheus

influx
USE prometheus
select * from /.*/ limit 1
```

