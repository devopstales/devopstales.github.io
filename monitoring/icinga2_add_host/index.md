---
title: Add host to Icinga2
url: https://devopstales.github.io/monitoring/icinga2_add_host/
date: 2019-12-11
keywords: icinga, icinga2, nagios
---


In this tutorial I will show you how to add hosts to Icinga2.

<!--more-->

### Create new host
```
nano /etc/icinga2/conf.d/my_router.conf
object Host "Router" {
  address = "192.168.1.1"
  check_command = "hostalive"
}

```

If we want to check the web admin interface of the router we can add a http check to see if the HTTP server is alive and responds with the proper HTTP codes

```
nano /etc/icinga2/conf.d/my_router.conf
...
object Service "http" {
  host_name = "Router"
  check_command = "http"
}
```

### Ad custom check

We will us the check_udpport script. This is the syntax.
```
/usr/lib64/nagios/plugins/check_udpport –H 127.0.0.1 –p 69
```

Now we need to create a nwe command in icinga2:
```
nano /etc/icinga2/conf.d/commands.conf
object CheckCommand "myudp" {
  command = [ PluginDir + "/check_udpport" ]

    arguments = {
    "-H" = "$addr$"
    "-p" = "$port$"
  }
  vars.addr = "$address$"
}
```

We have a nwe command so we can create a service to use rhis command:

```
nano /etc/icinga2/conf.d/my_router.conf
...
object Service "dhcp" {
  host_name = Router"
  check_command = "myudp"
vars.port = "67"
}
```

Test the config and restart:
```
icinga2 daemon -C
systemctl restart icinga2
```

