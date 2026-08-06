---
title: Install nrpe tp Icinga2
url: https://devopstales.github.io/monitoring/icinga2_nrpe/
date: 2019-12-12
keywords: icinga, icinga2, nagios, nrpe
---


In this tutorial I will show you how to nrpe check in Icinga2.

<!--more-->

### install nrpe on the clients
```
yum install nrpe nagios-plugins-all -y

firewall-cmd --permanent --add-port=5665/tcp
firewall-cmd --reload
```

### install nrpe on the server
```
yum install nrpe nagios-plugins-all -y

firewall-cmd --permanent --add-port=5665/tcp
firewall-cmd --reload
```

### configurate icinga to use nrpe
```
vi /etc/icinga2/conf.d/linux_services.conf
apply Service "nrpe-disk-root" {
  import "generic-service"
  check_command = "nrpe"
  vars.nrpe_command = "check_disk"
  vars.nrpe_arguments = [ "20%", "10%", "/" ]
  assign where "linux-servers" in host.groups
  ignore where match("*icinga*", host.name)
}

vi /etc/icinga2/conf.d/CLIENTS/server1.conf
object Host "server1" {
  address = "192.168.10.60"
  check_command = "ping"
  vars.os = "Linux"
}
```

Test the config and restart:
```
icinga2 daemon -C
systemctl restart icinga2
```

