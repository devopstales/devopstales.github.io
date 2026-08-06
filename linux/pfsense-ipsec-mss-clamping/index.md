---
title: pfsense: IPSec SSH connectivity issue
url: https://devopstales.github.io/linux/pfsense-ipsec-mss-clamping/
date: 2022-09-28
keywords: pfsense, IPSec, VPN, ssh
---


In this post I will setup an IPSec dynamic route-based vpn tunnel between two pfSense Appliances.
<!--more-->

### Main situation

I have two Pfsense firewalls for two sites. Sites are connected to each other with Pfsense IPsec tunnel. I experienced a strange issue, I can’t ssh from one site to a vm to the noter.

```bash
$ telnet 192.168.3.1  22
Trying 192.168.3.1...
Connected to 192.168.3.1.
Escape character is '^]'.
SSH-2.0-OpenSSH_7.4
```

I can get SSH banner with telnet but does not work. 

I've analyzed this a bit and with wireshark there are a lot of TCP - Retransmissions


![TCP - Retransmissions](/img/include/pfsense_ipsec_mss_01.webp)

If I do a ping with Packet Size of 969 Byte everythin is okay, with 970 there is packetloss.

```bash
ping -f 192.168.3.1 -l  969
```

### The solution

So there is a Issue with fragmentation. I enabled **Maximum MMS** in **VPN->IPSec->Advanced** Settings and set value to **1350**

![MSS Config](/img/include/pfsense_ipsec_mss_02.webp)

With the upgrade of Pfsense 2.6 this menu is change. It is in **System->Advanced->Firewall & NAT**

![MSS Config](/img/include/pfsense_ipsec_mss_03.webp)





