---
title: Install Wazuh SIEM
url: https://devopstales.github.io/linux/wazuh-siem/
date: 2023-09-01
keywords: Wazuh, SIEM, logging, CentOS
---


In this post I will show you how to install and configure the Wazuh Open Source SIEM solution.
<!--more-->

### What is a SIEM?

Security information and event management, or SIEM, is a security solution software that helps organizations recognize and address potential security threats and vulnerabilities before they have a chance to disrupt business operations. 

### What is Wazuh?

Wazuh is a security platform that provides unified XDR and SIEM protection for endpoints and cloud workloads. The solution is composed of a single universal agent and three central components: the Wazuh server, the Wazuh indexer, and the Wazuh dashboard. For more information, check the Getting Started documentation.

Wazuh is free and open source. Its components abide by the GNU General Public License, version 2, and the Apache License, Version 2.0 (ALv2).

![wazuh infra](/img/include/wazuh-architecture.webp)

### Install wazuh-indexer

Generating the SSL certificates:

```bash
curl -sO https://packages.wazuh.com/4.5/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.5/config.yml
```

```bash
nano config.yml
nodes:
  # Wazuh indexer nodes
  indexer:
    - name: wazuh.mydomain.intra
      ip: 192.168.10.50

  server:
    - name: wazuh.mydomain.intra
      ip: 192.168.10.50

  # Wazuh dashboard nodes
  dashboard:
    - name: wazuh.mydomain.intra
      ip: 192.168.10.50
```

Run `./wazuh-certs-tool.sh` to create the certificates.

```bash
bash ./wazuh-certs-tool.sh -A

tar -cvf ./wazuh-certificates.tar -C ./wazuh-certificates/ .
```

Install packages:

```bash
yum install coreutils

rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH
echo -e '[wazuh]\ngpgcheck=1\ngpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH\nenabled=1\nname=EL-$releasever - Wazuh\nbaseurl=https://packages.wazuh.com/4.x/yum/\nprotect=1' | tee /etc/yum.repos.d/wazuh.repo

yum -y install wazuh-indexer
```

Configure wazuh-indexer:

```bash
nano /etc/wazuh-indexer/opensearch.yml
network.host: "192.168.10.50"
node.name: "wazuh.mydomain.intra"
cluster.initial_master_nodes:
- "wazuh.mydomain.intra"
...
plugins.security.nodes_dn:
- "CN=wazuh.mydomain.intra,OU=Wazuh,O=Wazuh,L=California,C=US"
```

Deploying certificates:

```bash
export NODE_NAME=wazuh.mydomain.intra

mkdir /etc/wazuh-indexer/certs
tar -xf ./wazuh-certificates.tar -C /etc/wazuh-indexer/certs/ ./$NODE_NAME.pem \
./$NODE_NAME-key.pem ./admin.pem ./admin-key.pem ./root-ca.pem
mv -n /etc/wazuh-indexer/certs/$NODE_NAME.pem /etc/wazuh-indexer/certs/indexer.pem
mv -n /etc/wazuh-indexer/certs/$NODE_NAME-key.pem /etc/wazuh-indexer/certs/indexer-key.pem
chmod 500 /etc/wazuh-indexer/certs
chmod 400 /etc/wazuh-indexer/certs/*
chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs
```

Configure memory heap size:

```bash
echo "bootstrap.memory_lock: true" >> /etc/wazuh-indexer/opensearch.yml

mkdir /etc/systemd/system/wazuh-indexer.service.d/

echo "[Service]
LimitMEMLOCK=infinity
" > /etc/systemd/system/wazuh-indexer.service.d/override.conf

nano /etc/wazuh-indexer/jvm.options
-Xms4g
-Xmx4g
```

Starting the service:

```bash
systemctl daemon-reload
systemctl enable wazuh-indexer
systemctl start wazuh-indexer
```

Cluster initialization:

```bash
/usr/share/wazuh-indexer/bin/indexer-security-init.sh

curl -k -u admin:admin https://192.168.10.50:9200

curl -k -u admin:admin https://192.168.10.50:9200/_cat/nodes?v
```

### Install wazuh-manager

Install the package:

```bash
yum -y install wazuh-manager
```

Enable and start the Wazuh manager service.

```bash
systemctl daemon-reload
systemctl enable wazuh-manager
systemctl start wazuh-manager
```

### Installing Filebeat

Install the Filebeat package:

```bash
yum -y install filebeat
```

Configuring Filebeat:

```bash
curl -so /etc/filebeat/filebeat.yml https://packages.wazuh.com/4.5/tpl/wazuh/filebeat/filebeat.yml

nano /etc/filebeat/filebeat.yml
# Wazuh - Filebeat configuration file
 output.elasticsearch:
 hosts: ["192.168.10.50:9200"]
 protocol: https
 username: ${username}
 password: ${password}
```

Create a Filebeat keystore to securely store authentication credentials:

```bash
filebeat keystore create

echo admin | filebeat keystore add username --stdin --force
echo admin | filebeat keystore add password --stdin --force
```

Download and apply wazuh index template to the indexer:

```bash
curl -so /etc/filebeat/wazuh-template.json https://raw.githubusercontent.com/wazuh/wazuh/4.5/extensions/elasticsearch/7.x/wazuh-template.json
chmod go+r /etc/filebeat/wazuh-template.json
```

Install the certificates and the wazuh module to filebeat:

```bash
curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.2.tar.gz | tar -xvz -C /usr/share/filebeat/module

mkdir /etc/filebeat/certs
tar -xf ./wazuh-certificates.tar -C /etc/filebeat/certs/ ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./root-ca.pem
mv -n /etc/filebeat/certs/$NODE_NAME.pem /etc/filebeat/certs/filebeat.pem
mv -n /etc/filebeat/certs/$NODE_NAME-key.pem /etc/filebeat/certs/filebeat-key.pem
chmod 500 /etc/filebeat/certs
chmod 400 /etc/filebeat/certs/*
chown -R root:root /etc/filebeat/certs
```

Starting the Filebeat service

```bash
systemctl daemon-reload
systemctl enable filebeat
systemctl start filebeat

filebeat test output
```

### Install Wazuh-dashboard

Installing packages:

```bash
yum install libcap

yum -y install wazuh-dashboard
```

Configuring the Wazuh dashboard:

```bash
nano /etc/wazuh-dashboard/opensearch_dashboards.yml
server.host: 0.0.0.0
   server.port: 443
   opensearch.hosts: https://localhost:9200
   opensearch.ssl.verificationMode: certificate
```

Deploying certificates:

```bash
export NODE_NAME=wazuh.mydomain.intra

export NODE_NAME=wazuh.mydomain.intra

mkdir /etc/wazuh-dashboard/certs
tar -xf ./wazuh-certificates.tar -C /etc/wazuh-dashboard/certs/ ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./root-ca.pem
mv -n /etc/wazuh-dashboard/certs/$NODE_NAME.pem /etc/wazuh-dashboard/certs/dashboard.pem
mv -n /etc/wazuh-dashboard/certs/$NODE_NAME-key.pem /etc/wazuh-dashboard/certs/dashboard-key.pem
chmod 500 /etc/wazuh-dashboard/certs
chmod 400 /etc/wazuh-dashboard/certs/*
chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs
```

Starting the Wazuh dashboard service:

```bash
systemctl daemon-reload
systemctl enable wazuh-dashboard
systemctl start wazuh-dashboard
```

Access the Wazuh web interface with your credentials: Go to `https://192.168.10.50` and logi with user `admin` and password `admin`.

Use the Wazuh passwords tool to change all the internal users' passwords.

>> Save the passwords for later use. This is the only time the script shows it.

```bash
/usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh --change-all --admin-user wazuh --admin-password wazuh

INFO: The password for user admin is yWOzmNA.?Aoc+rQfDBcF71KZp?1xd7IO
INFO: The password for user kibanaserver is nUa+66zY.eDF*2rRl5GKdgLxvgYQA+wo
INFO: The password for user kibanaro is 0jHq.4i*VAgclnqFiXvZ5gtQq1D5LCcL
INFO: The password for user logstash is hWW6U45rPoCT?oR.r.Baw2qaWz2iH8Ml
INFO: The password for user readall is PNt5K+FpKDMO2TlxJ6Opb2D0mYl*I7FQ
INFO: The password for user snapshotrestore is +GGz2noZZr2qVUK7xbtqjUup049tvLq.
WARNING: Wazuh indexer passwords changed. Remember to update the password in the Wazuh dashboard and Filebeat nodes if necessary, and restart the services.
INFO: The password for Wazuh API user wazuh is JYWz5Zdb3Yq+uOzOPyUU4oat0n60VmWI
INFO: The password for Wazuh API user wazuh-wui is +fLddaCiZePxh24*?jC0nyNmgMGCKE+2
INFO: Updated wazuh-wui user password in wazuh dashboard. Remember to restart the service.
```

