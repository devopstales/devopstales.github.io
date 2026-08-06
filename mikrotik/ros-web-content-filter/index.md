---
title: MikroTik - RouterOS: Web Content Filter
url: https://devopstales.github.io/mikrotik/ros-web-content-filter/
date: 2022-07-14
keywords: mikrotik, routeros, Web Content Filter, firewall, router
---


In this post I will show you how you can can filter web content with MikroTik RouterOS router.
<!--more-->

Sometimes you may want to block certain websites, for example, deny access to entertainment sites for employees, deny access to porn, and so on. This can be achieved by redirecting HTTP traffic to a proxy server and use an access-list to allow or deny certain websites.

First, we need to add a NAT rule to redirect HTTP to our proxy. We will use RouterOS built-in proxy server running on port 8080.

```bash
/ip firewall nat
  add chain=dst-nat protocol=tcp dst-port=80 src-address=192.168.88.0/24 \
    action=redirect to-ports=8080
```

Enable web proxy and drop some websites:

```bash
/ip proxy set enabled=yes
/ip proxy access add dst-host=www.facebook.com action=deny
/ip proxy access add dst-host=*.youtube.* action=deny
/ip proxy access add dst-host=:vimeo action=deny
```

### L7 Filtering

There is a different method called Layer 7 filtering. It use regular expression matches:

```bash
/ip firewall layer7-protocol add name=torrentsites regexp="^.*(get|GET).+(torrent|\
thepiratebay|isohunt|entertane|demonoid).*\$\"
```

Drop connection to this sites:

```bash
/ip firewall filter add chain=forward src-address=192.168.88.0/24 layer7-protokol=torrentsites \
action=drop comment=torrentsites

/ip firewall filter add chain=forward src-address=192.168.88.0/24 protokol=l7 dst-port=53 \
layer7-protokol=torrentsites action=drop comment=torrentsitesDropDNS

/ip firewall filter add chain=forward src-address=192.168.88.0/24 content=torrent \
action=drop comment=torrent_drop

/ip firewall filter add chain=forward src-address=192.168.88.0/24 content=tracker \
action=drop comment=tracker_drop

/ip firewall filter add chain=forward src-address=192.168.88.0/24 content=getpeer \
action=drop comment=getpeer_drop

/ip firewall filter add chain=forward src-address=192.168.88.0/24 content=info_hash \
action=drop comment=info_hash_drop

/ip firewall filter add chain=forward src-address=192.168.88.0/24 content=announce_peers \
action=drop comment=announce_peers_drop
```

### DNS Poisoning

The third method is to use a dns server that block harmful contents to resolve:

```bash
# configure the dns sever
/ip dns set servers=195.46.39.39,195,46,39,40

# Intercept all the dns requests and redirect to RouterOS
/ip firewall filter add action=dst-nat chain=dstnat dst-port=53 in-interface=ether2 protocol=tcp to-address=192.168.88.1 to-port=53
/ip firewall filter add action=dst-nat chain=dstnat dst-port=53 in-interface=ether2 protocol=udp to-address=192.168.88.1 to-port=53
```
