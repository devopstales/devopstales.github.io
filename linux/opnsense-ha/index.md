---
title: Configurate HA opnsense cluster
url: https://devopstales.github.io/linux/opnsense-ha/
date: 2019-05-24
keywords: opnsense, HA, cluster
---


In this post I will configure 2 opnsense server to a HA cluster.
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

### Firewall rules For sync
On both firewalls add two rules to allow traffic on the SYNC interface: <br>
go to `Firewall > Rules > Sync` and click `Add`.

Rule 1:
![Example image](/img/include/opnsense_carp1.webp)

Rule 2:
![Example image](/img/include/opnsense_carp2.webp)

Rule 3:
![Example image](/img/include/opnsense_carp3.webp)

### Synchronization Settings
Go to `System > High Availalility > Settings`. Configure the sections like on the pictures.

Master:
![Example image](/img/include/opnsense_carp4.webp)

Slave:
![Example image](/img/include/opnsense_carp5.webp)

Test the synchronisation. Go to `System > User management` and createa new user on the master node. <br>
Then check on the slave node.

If it doesn't work, check:

* Are the firewall web interfaces running on the same protocols and ports?
* Is the admin password set correctly? (User Manager > Users > admin.)
* Are the firewall rules to allow synch set to use the correct interface (SYNC)?
* If you're using VMs, are the firewalls on the same internal network?

### create virtual IPs
On the master node go to` Firewall > Virtual IPs > Settings` and click Add. Create a new VIP adres for LAN and WAN interfaces.

WAN VIP on master:
![Example image](/img/include/opnsense_carp6.webp)

LAN VIP on master:
![Example image](/img/include/opnsense_carp7.webp)

### Change outbound NAT
Change the configuration of the outbound NAT to use the shared public IP (the WAN VIP) <br>
Go to `Firewall > NAT > Outbound` and set the mode to Hybrid Outbound NAT rule generation. <br> <br>
Create a new Outbound rule like this: <br>
![Example image](/img/include/opnsense_carp8.webp)

The translatino / target must be the WANIP IP. <br>
It should end up looking like this: <br>

![Example image](/img/include/opnsense_carp9.webp)

If you’ll be using your opnsense firewall as a DNS resolver you must change the settings of the DNS service (`Services > DNS Resolver > General Settings`) to lissen on the LAN VIP address. Then chnage the address of the DNS server in the DHCP configuration to us the LAN VIP adress.

