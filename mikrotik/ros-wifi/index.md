---
title: MikroTik - RouterOS: Basic Wifi Config
url: https://devopstales.github.io/mikrotik/ros-wifi/
date: 2022-07-16
keywords: mikrotik, routeros, wifi, firewall, router
---


In this post I will show you how to configure a basic wifi on MikroTik RouterOS router.
<!--more-->

For ease of use bridged wireless setup will be made so that your wired hosts are in the same Ethernet broadcast domain as wireless clients.

```bash
/ip address remove ether2
/interface bridge add name=local
/interface bridge port add interface=ether2 bridge=local
/ip address add address=192.168.88.1/24 interface=local
```

The important part is to make sure that our wireless is protected, so the first step is the security profile.

```bash
/interface wireless security-profiles
  add name=myProfile authentication-types=wpa2-psk mode=dynamic-keys \
    wpa2-pre-shared-key=1234567890
```

Now when the security profile is ready we can enable the wireless interface and set the desired parameters:

```bash
/interface wireless
  enable wlan1;
  set wlan1 band=2ghz-b/g/n channel-width=20/40mhz-Ce distance=indoors \
    mode=ap-bridge ssid=MikroTik-006360 wireless-protocol=802.11 \
    security-profile=myProfile frequency-mode=regulatory-domain \
    set country=latvia antenna-gain=3
```

The last step is to add a wireless interface to a local bridge, otherwise connected clients will not get an IP address:

```bash
/interface bridge port
  add interface=wlan1 bridge=local
```

