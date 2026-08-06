---
title: Install telegraf on pfsense
url: https://devopstales.github.io/monitoring/pfsense-telegraf/
date: 2019-03-06
keywords: pfsense, prometheus, telegraf
---


Install and configure telegraf on pfsense to provides system information to prometheus.

<!--more-->

### Install telegraf
* At the System / Package menu install the telegraf service to the pfsense.
* ssh to the pfsense server and open a shell

![Example image](/img/include/pfsense-telegraf.webp)

### Install nano
```
pkg
pkg update
pkg install nano
```


### Configure telegraf
```
cd /usr/local/etc

nano telegraf.conf
[[outputs.prometheus_client]]
 listen = ":9273"

echo "telegraf_enable="YES"" >> /etc/rc.conf

cd /usr/local/etc/rc.d
./telegraf restart
```

