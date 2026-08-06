---
title: Sending syslog via Kafka into Graylog
url: https://devopstales.github.io/linux/graylog_kafka/
date: 2021-03-20
keywords: Graylog, Kafka, rsyslog, logging, CentOS
---


Graylog supports Apache Kafka as a transport for various inputs such as GELF, syslog, and Raw/Plaintext inputs. The Kafka topic can be filtered by a regular expression and depending on the input, various additional settings can be configured.
<!--more-->

### Requirements

* Running graylog server

### Installing Apache Kafka in CentOS 7

```
yum install -y java-1.8.0-openjdk-headless.x86_64

nano /etc/profile
export JRE_HOME=/usr/lib/jvm/jre
export JAVA_HOME=/usr/lib/jvm/jre-1.8.0-openjdk
PATH=$PATH:$JRE_HOME:$JAVA_HOME

source /etc/profile
```

```
useradd kafka -m
sudo usermod -aG wheel kafka

wget https://downloads.apache.org/kafka/2.7.0/kafka_2.13-2.7.0.tgz -O kafka_2.13-2.7.0.tgz
tar -xzf kafka_2.13-2.7.0.tgz
mv kafka_*/ /opt/kafka
chown kafka:kafka -R /opt/kafka/
```

```
nano /etc/systemd/system/zookeeper.service
[Unit]
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=simple
User=kafka
ExecStart=/opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
ExecStop=/opt/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

```
nano /etc/systemd/system/kafka.service
[Unit]
Requires=network.target remote-fs.target zookeeper.service
After=network.target remote-fs.target zookeeper.service

[Service]
Type=simple
User=kafka
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

```
nano /opt/kafka/config/server.properties
listeners=PLAINTEXT://:9092
log.dirs=/var/log/kafka-logs

sudo mkdir -p /var/log/kafka-logs
chown kafka:kafka -R /var/log/kafka-logs

systemctl daemon-reload
systemctl start zookeeper.service
systemctl start kafka.service
systemctl enable zookeeper.service
systemctl enable kafka.service
systemctl status zookeeper.service
systemctl status kafka.service
```

### Create kafka topic
```
/opt/kafka/bin/kafka-topics.sh --create \
--zookeeper localhost:2181 \
--replication-factor 1 \
--partitions 1 \
--topic logs

/opt/kafka/bin/kafka-topics.sh \
--zookeeper localhost:2181 \
--list
```

### Install rsyslog
```
yum install -y rsyslog rsyslog-kafka
```

```
nano /etc/rsyslog.d/kafka.conf
:omusrmsg:PreserveFQDN on
template(name="ls_json"
         type="list"
         option.json="on") {
           constant(value="{")
             constant(value="\"timestamp\":\"")     property(name="timereported" dateFormat="rfc3339")
             constant(value="\",\"@version\":\"1")
             constant(value="\",\"message\":\"")     property(name="msg")
             constant(value="\",\"source\":\"")        property(name="hostname")
             constant(value="\",\"severity\":\"")    property(name="syslogseverity-text")
             constant(value="\",\"facility\":\"")    property(name="syslogfacility-text")
             constant(value="\",\"programname\":\"") property(name="programname")
             constant(value="\",\"procid\":\"")      property(name="procid")
           constant(value="\"}\n")
         }

$ModLoad omkafka
*.warning action(type="omkafka" topic="logs" broker=["192.168.0.110:9092"] template="ls_json" errorfile="/var/log/rsyslog-kafka.err")
```

```
systemctl restart rsyslog

netstat -nputw | grep 9092 | grep rsyslog
tcp        0      0 192.168.0.110:50912     192.168.0.110:9092      ESTABLISHED 5816/rsyslogd       
tcp        0      0 127.0.0.1:33624         127.0.1.1:9092          ESTABLISHED 5816/rsyslogd

# List content in topic:
/opt/kafka/bin/kafka-console-consumer.sh \
--topic logs --from-beginning \
--bootstrap-server localhost:9092
```

### Create input in Graylog
Go to `System > Inputs` and launch a new `Raw/Plaintext Kafka Input`. 

```
Title: kafka
Legacy mode: false
Bootstrap Servers(optional): 127.0.0.1:9092
Consumer group id(optional): graylog2
```

Then create an JSON extractor on message field.

