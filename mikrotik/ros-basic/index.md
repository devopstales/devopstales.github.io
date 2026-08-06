---
title: MikroTik - RouterOS: Basic configuration
url: https://devopstales.github.io/mikrotik/ros-basic/
date: 2022-07-10
keywords: mikrotik, routeros, statik ip, pppoe, internet
---


In this post I will show you the basic configuration of a MikroTik RouterOS router.
<!--more-->

## Connecting to the Router
There are two types of routers:

* With default configuration
* Without default configuration. When no specific configuration is found, IP address 192.168.88.1/24 is set on ether1 or combo1, or sfp1.

## Connect to the Router

You cen connect to the router with tree differente mode:

* Web conssole
* WinBox client discowers and connect to MikroTik RouterOS by mac-adress or IP
* ssh connection

As you see there is several options to connect and configure, but here we will use one method that suits our needs.

Connect Routers `ether1` port to the `WAN` cable and connect your `PC` to `ether2`.  Now open WinBox and look for your router in neighbor discovery. 

![WinBox neighbor discovery](img/include/ros_basic_01.webp)

If you see the router in the list, click on MAC address and click Connect.

After connection open a terminal with the `New Termonal` menu and reset the configuration:

```bash
/system reset-configuration no-defaults=yes skip-backup=yes
```

### Set admin password

Every Router has a factory preconfigured user `admin` with an `empty` password. To set the passford `Password1` to user `admin` Use the command from terminal:

```bash
user set admin password=Password1
```

## Configuring IP Access

Since MAC connection is not very stable, the first thing we need to do is to set up a router so that IP connectivity is available:

```bash
/ip address add address=192.168.88.1/24 interface=ether2
```

## RouterOS license keys

MikroTik hardware routers that run RouterOS come preinstalled with a RouterOS license, if you have purchased a RouterOS based device, nothing must be done regarding the license.

Licensing information can be read from CLI system console:

```bash
/system license print

    software-id: "43NU-NLT9"
         nlevel: 6
       features: 
```

## Configuring Internet Connection

The next step is to get internet access to the router. There can be several types of internet connections, but the most common ones are:

* dynamic public IP address;
* static public IP address;
* PPPoE connection.

### Dynamic Public IP

Configure a DHCP client to get ip from a DHCP server:

```bash
/ip dhcp-client add disabled=no interface=ether1 comment=WAN
```

### Static Public IP

If ther is no DHCP server in the network you can configure a static ip address and DNS server:

```bash
# static ip
/ip address add address=1.2.3.100/24 interface=ether1 comment=WAN

# Default Gateway
/ip route add gateway=1.2.3.1

# Configure dns server
/ip dns set servers=8.8.8.8
```

### PPPoE Connection

PPPoE connection also gives you a dynamic IP address and can configure dynamically DNS and default gateway. Typically service provider (ISP) gives you a username and password for the connection

```bash
/interface pppoe-client
  add disabled=no interface=ether1 user=admin password=Password1 \
    add-default-route=yes use-peer-dns=yes comment=WAN
```

### Verify Connectivity

```bash
ip address print

Flags: D - DYNAMIC
Columns: ADDRESS, NETWORK, INTERFACE
#   ADDRESS           NETWORK       INTERFACE
0   192.168.88.1/24   192.168.88.0  ether2
1 D 10.0.2.15/24      10.0.2.0      ether1
```

```bash
ping 8.8.8.8

  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 8.8.8.8                                    56 254 37ms402us
    1 8.8.8.8                                    56 254 4ms978us
    2 8.8.8.8                                    56 254 4ms992us
    3 8.8.8.8                                    56 254 4ms97us
    sent=4 received=4 packet-loss=0% min-rtt=4ms97us avg-rtt=12ms867us max-rtt=37ms402us
```

### How to change MikroTik RouterOS names

```bash
[vagrant@MikroTik] > system identity print
  name: MikroTik

[vagrant@MikroTik] > system identity set name=ros01.mydomain.intra

[vagrant@ros01.mydomain.intra] > system identity print
  name: ros01.mydomain.intra
```

### Time Server Configuration

```bash
system ntp client print 
     enabled: no
        mode: unicast
     servers: 
  freq-drift: 0 PPM
      status: stopped

system clock print 
                  time: 10:30:15
                  date: jul/17/2022
  time-zone-autodetect: yes
        time-zone-name: manual
            gmt-offset: +00:00
            dst-active: no
```

```bash
system clock set time-zone-autodetect=no
system clock set time-zone-name=CET

system clock print
                  time: 12:36:54
                  date: jul/17/2022
  time-zone-autodetect: no
        time-zone-name: CET
            gmt-offset: +02:00
            dst-active: yes
```

```bash
# RouterOS 7
system ntp client set enabled=yes
system/ntp/client/servers/add address=2001:4860:4860::8844
system/ntp/client/servers/add address=202.4.100.106

# RouterOS 6
system ntp client set enabled=yes primary-ntp=2001:4860:4860::8844 secondary-ntp=202.4.100.106
```
