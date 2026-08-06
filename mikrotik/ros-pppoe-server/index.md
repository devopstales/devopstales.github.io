---
title: MikroTik - RouterOS: PPPOE Server
url: https://devopstales.github.io/mikrotik/ros-pppoe-server/
date: 2022-07-17
keywords: mikrotik, routeros, PPPOE, pfsense, router
---


In this post I will show you how to configure a PPPOE Server on on MikroTik RouterOS router.
<!--more-->

### Architecture

```bash
       | NAT Network
  -----------
  | RuterOS |
  -----------
Host only |  \
              \
               \
       DHCP     \
Internet for NAT \
Internal Network  |
             -----------
	     | pfsense |
	     -----------
```

```bash
# base		 
int print
int set 0 name=WAN
int set 1 name=MANAGEMENT
int set 2 name=LAN 
```

```bash
# enable dhcp client			 
ip dhcp-client add interface=WAN disabled=no
ip dhcp-client add interface=MANAGEMENT disabled=no
ip dhcp-client print detail

# outbound nat for interget
ip firewall nat add chain=srcnat action=masquerade out-interface=WAN
```

```bash
# set static ip
ip address add address=10.10.10.1/24 interface=LAN
ip address print

# configure ros as dns server
ip dns set allow-remote-requests=yes servers=8.8.8.8
```

```bash
# add dhcp server on LAN
ip dhcp-server setup   
Select interface to run DHCP server on 

dhcp server interface: LAN
Select network for DHCP addresses 

dhcp address space: 10.10.10.0/24
Select gateway for given network 

gateway for dhcp network: 10.10.10.1
Select pool of ip addresses given out by DHCP server 

addresses to give out: 10.10.10.50-10.10.10.100 
Select DNS servers 

dns servers: 8.8.8.8,8.8.4.4
Select lease time 

lease time: 1d
```

```bash
# pppoe
ip pool add name=pppoe-pool ranges=10.10.20.10-10.10.20.100

ppp profile add name=pppoe-profile local-address=10.10.20.1 \
remote-address=pppoe-pool dns-server=10.10.20.1 rate-limit=512k/6512k

interface pppoe-server server add interface=LAN \
max-mtu=1488 max-mru=1488 \
keepalive-timeout=disabled one-session-per-host=yes \
max-sessions=0 default-profile=pppoe-profile \
authentication=pap disabled=no

ip firewall nat add chain=srcnat src-address=10.10.20.0/24 action=masquerade

ppp secret add name=tester password=Password1 profile=pppoe-profile out-interface=WAN
```

