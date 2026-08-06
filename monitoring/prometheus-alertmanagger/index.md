---
title: Install Alertmanagger
url: https://devopstales.github.io/monitoring/prometheus-alertmanagger/
date: 2018-10-18
keywords: prometheus, alerta, nosql
---


The Alertmanager handles alerts sent by client applications such as the Prometheus server.

<!--more-->

### Download Alertmanager
```
wget https://github.com/prometheus/alertmanager/releases/download/v0.15.0-rc.1/alertmanager-0.15.0-rc.1.linux-amd64.tar.gz
tar -xzf alertmanager-0.15.0-rc.1.linux-amd64.tar.gz
```

### Install binaris
```
useradd --no-create-home --shell /bin/false alertmanager

mkdir /etc/alertmanager
mkdir /etc/alertmanager/template
mkdir -p /var/lib/alertmanager/data
touch /etc/alertmanager/alertmanager.yml

chown -R alertmanager:alertmanager /etc/alertmanager
chown -R alertmanager:alertmanager /var/lib/alertmanager

cp alertmanager-*linux-amd64/alertmanager /usr/local/bin/
cp alertmanager-*linux-amd64/amtool /usr/local/bin/

chown alertmanager:alertmanager /usr/local/bin/alertmanager
chown alertmanager:alertmanager /usr/local/bin/amtool
```

### Create servis for Alertmanager
```
nano /etc/systemd/system/alertmanager.service
[Unit]
Description=Prometheus Alertmanager Service
Wants=network-online.target
After=network.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
    --config.file /etc/alertmanager/alertmanager.yml \
    --storage.path /var/lib/alertmanager/data
Restart=always

[Install]
WantedBy=multi-user.target
```

### Configure Alertmanager
```
nano /etc/alertmanager/alertmanager.yml
global:
  smtp_smarthost: 'localhost:25'
  smtp_from: 'alertmanager@devopstales.intra'
#  smtp_auth_username: 'alertmanager'
#  smtp_auth_password: 'password'

templates:
- '/etc/alertmanager/template/*.tmpl'

route:
  repeat_interval: 3h
  receiver: mails

receivers:
- name: 'mails'
  email_configs:
  - to: 'admin@devopstales.intra'
```

### Configure Prometheus
```
nano /etc/prometheus/prometheus.yml
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

alerting:
  alertmanagers:
  - static_configs:
    - targets:
       - localhost:9093
```

```
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
sudo systemctl ststus alertmanager
```

