---
title: MikroTik - RouterOS: Firewall configurations
url: https://devopstales.github.io/mikrotik/ros-firewall/
date: 2022-07-16
keywords: mikrotik, routeros, firewall, router
---


In this post I will show you how to configure the firewall on on MikroTik RouterOS router.
<!--more-->

### NAT Configuration

By default the compuers on the LAN network are not yet able to access the Internet, because locally used addresses are not routable over the Internet. The solution for this problem is to change the source address for outgoing packets to routers public IP. This can be done with the NAT rule:

```bash
/ip firewall nat
  add chain=srcnat out-interface=ether1 action=masquerade
```

### Port Forwarding

Some client devices may need to accassable directy from the internet like a webserver:

```bash
# by interface
/ip firewall nat 
add chain=dstnat in-interface=ether1 protocol=tcp dst-port=80 \
action=dst-nat to-address=192.168.88.50 to-ports=80

# by public address
/ip firewall nat 
add chain=dstnat dst-address=<your-public-ip-address-here> protocol=tcp dst-port=443 \
  action=dst-nat to-address=192.168.88.50
```

### Firewall Rules

