---
title: WAN failower on pfsense
url: https://devopstales.github.io/linux/pfsense-wlan/
date: 2019-04-12
keywords: pfsense, HA, cluster
---


In this pos I will create a WAN failower configuration.
<!--more-->

### The Architecture
```

------- WAN1 ------
| ----- WAN2 ---- |
| |             | |
PF1 -- sync -- PF2
 |               |
 ----- LAN -------  


WAN1: 192.168.0.0/24 (Bridgelt)
LAN: 10.0.1.0/24
SYNC: 10.0.2.0/24
WAN2: 10.0.4.0/24
```



### Configurate WIP for WAN2
At `Firewall > Virtual IPs > Add`
![Example image](/img/include/pfsenseWAN_1.webp)


### Add Gateway for WAN interfaces
At `System > Routing > Add`
![Example image](/img/include/pfsenseWAN_2.webp)

### Configuring Monitor IP
At` System > Routing > Edit gateways` and add google dns ad monitoring ip
![Example image](/img/include/pfsenseWAN_3.webp)

![Example image](/img/include/pfsenseWAN_4.webp)

### Configuring Gateway Group
At` System > Routing > Gateway Groups` Create 3 Groups
![Example image](/img/include/pfsenseWAN_5.webp)

![Example image](/img/include/pfsenseWAN_6.webp)

![Example image](/img/include/pfsenseWAN_7.webp)

### Configuring Firewall Rules
Got to `Firewall > Rules > LAN` and edit the IPv4 rule. Chane the Gateway

![Example image](/img/include/pfsenseWAN_8.webp)

![Example image](/img/include/pfsenseWAN_9.webp)

Clone the changed roles to two other rules and change the Gateway to the other Gateway Groups.
![Example image](/img/include/pfsenseWAN_10.webp)

### Configurate NAT
Go to` Firewall > NAT > Outbound` <br>
Clone WAN1 rules and edit them to WLAN2
![Example image](/img/include/pfsenseWAN_11.webp)

![Example image](/img/include/pfsenseWAN_12.webp)

## pfSense email notification when WAN connection goes down
Go to `System > Advanced > Notifications`

### Example with Google Gmail SMTP
![Example image](/img/include/pfsenseWAN_13.webp)

