---
title: Configurate ipmitool
url: https://devopstales.github.io/linux/ipmitool-config/
date: 2019-04-29
keywords: Supermicro, ipmitool, pfsense, telegraf
---


The Intelligent Platform Management Interface (IPMI) is a set of computer interface specifications for an autonomous computer subsystem that provides management and monitoring capabilities independently of the host CPU, firmware (BIOS or UEFI) and operating system.
<!--more-->

### Ipmitool on pfsense
```
[2.3.3-RELEASE][root@fw.makz.me]/root: ipmitool
Could not open device at /dev/ipmi0 or /dev/ipmi/0 or /dev/ipmidev/0: No such file or directory

# solution
kldload ipmi
nano /boot/loader.conf
#Load ipmi.ko into the kernel
ipmi_load="YES"
```

### Ipmitool on Linux
```
modprobe ipmi_msghandler
modprobe ipmi_devintf
modprobe ipmi_si

nano /etc/modules
# OR
nano /etc/modprobe.d/*.conf
ipmi_msghandler
ipmi_devintf
ipmi_si
```

### Ipmitool Configuration
```
ipmitool lan set 1 ipsrc static
ipmitool lan set 1 ipaddr 192.168.0.211
ipmitool lan set 1 netmask 255.255.255.0
ipmitool lan set 1 defgw ipaddr 192.168.0.1
ipmitool lan set 1 arp respond on
ipmitool lan set 1 access on

ipmitool lan print 1
```

### Monitor ipmi with telegraf
```
nano /etc/telegraf/telegraf.conf
[[inputs.ipmi_sensor]]
        path = "/usr/bin/ipmitool"
        servers = ["username:password@lan(192.168.0.211)"]
        interval = "30s"
        timeout = "20s"
```

