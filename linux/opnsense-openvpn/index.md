---
title: Configure OpenVPN HA opnsense cluster
url: https://devopstales.github.io/linux/opnsense-openvpn/
date: 2019-05-25
keywords: opnsense, HA, cluster, OpenVPN
---


In this LAB I will be creating OpenVPN SSL Peer to Peer connection.
<!--more-->

### The Architecture
```
 ------ WAN ------
 |               |
PF1 -- sync -- PF2
 |               |
 ----- LAN -------  

WAN: 192.168.0.0/24 (Bridgelt)
LAN: 192.168.20.0/24
SYNC: 192.168.30.0/24
```

```
opn01:
WAN 192.168.0.28
LAN: 192.168.20.28
SYNC:192.168.30.28

opn02:
WAN 192.168.0.29
LAN: 192.168.20.29
SYNC:192.168.30.29
```

### Configurate the OpeVPN service
Got to `VPN > OpenVPN > Wizards`
![Example image](/img/include/opnsense_ovpn1.webp)

If you ulodad your certificate seledt that in the drop doew menu or select Add new Certificate to generate a new one.
![Example image](/img/include/opnsense_ovpn2.webp)

![Example image](/img/include/opnsense_ovpn3.webp)

Edit the Adwanced Configuration: <br>
![Example image](/img/include/opnsense_ovpn4.webp)

![Example image](/img/include/opnsense_ovpn5.webp)

![Example image](/img/include/opnsense_ovpn6.webp)

### Configurate NAT Rules to HA
Go to `Firewall > NAT > Outbound` and clone the manul LAN Rule
![Example image](/img/include/opnsense_ovpn8.webp)

### Enable Connection from OpenVPN to master and slave
In default there in no rout to the salve nod. <br>
 Go to `Firewll > Aliases > Add` and create alias for CARP members: <br>
![Example image](/img/include/opnsense_ovpn7.webp)


Then go back to `Firewall > NAT > Outbound > Settings` and create a new rule:
![Example image](/img/include/opnsense_ovpn9.webp)

