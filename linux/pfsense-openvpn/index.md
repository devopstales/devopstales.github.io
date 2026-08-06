---
title: Configure OpenVPN HA pfsense cluster
url: https://devopstales.github.io/linux/pfsense-openvpn/
date: 2019-04-11
---


In this LAB I will be creating OpenVPN SSL Peer to Peer connection.
<!--more-->

### Generating CA Certificate
At `System > Cert.Manager > CAs > Add`
![Example image](/img/include/OpenVPN_1.webp)

![Example image](/img/include/OpenVPN_2.webp)

### Generate Server Certificate
At `System > Cert.Manager > Certificates > Add`
![Example image](/img/include/OpenVPN_3.webp)

### Generate User Certificate
For this demo I will'create one certificate for all users, but in live you should create a separate certificate for all users.

At `System > Cert.Manager > Certificates > Add`
![Example image](/img/include/OpenVPN_4.webp)

At `SystemUser > ManagerUsers` add the User certificate for the users.
![Example image](/img/include/OpenVPN_5.webp)

### Intall Openvpn package exporter
Got to` System > Package Manager > Available Packages` and install `openvpn-client-export` plugin.

### Configurate the OpeVPN service
Got to `VPN > OpenVPN > Wizards`
![Example image](/img/include/OpenVPN_6.webp)

![Example image](/img/include/OpenVPN_7.webp)

![Example image](/img/include/OpenVPN_8.webp)

![Example image](/img/include/OpenVPN_9.webp)

Edit the Adwanced Configuration:
![Example image](/img/include/OpenVPN_18.webp)

![Example image](/img/include/OpenVPN_10.webp)

![Example image](/img/include/OpenVPN_11.webp)

### Configurate NAT Rules to HA
Go to `Firewall > NAT > Outbound` and clone the LAN Rules?
![Example image](/img/include/OpenVPN_12.webp)

![Example image](/img/include/OpenVPN_13.webp)

![Example image](/img/include/OpenVPN_14.webp)

![Example image](/img/include/OpenVPN_15.webp)

### Enable Connection from OpenVPN to master and slave
In default there in no rout to the salve nod. Go to `Firewll > Aliases > Add` and create alias for CARP members:
![Example image](/img/include/OpenVPN_16.webp)


Then go back to `Firewall > NAT > Outbound` and create a new rule:
![Example image](/img/include/OpenVPN_17.webp)

