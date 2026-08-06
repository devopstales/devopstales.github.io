---
title: Proxmox reload ifupdown2 network config from cli
url: https://devopstales.github.io/linux/proxmox-reload-network/
date: 2023-05-22
keywords: proxmox ve, proxmox cluster, ifupdown2
---


The correct way to remove nod from proxmox cluster.

<!--more-->

```bash
$ lsb_release -a
```

```bash
No LSB modules are available.
Distributor ID:	Debian
Description:	Debian GNU/Linux 11 (bullseye)
Release:	11
Codename:	bullseye
```

Install `ifupdown2` package which is a `ifupdown` re-written in Python.

```bash
apt get install ifupdown2
```

Display utility version.

```bash
$ ifreload --version
```

```bash
ifupdown2:3.1.0-1+pmx3
```

Inspect configuration for network interfaces.

```bash
$ cat /etc/network/interfaces
```

```bash
auto lo
iface lo inet loopback

iface enp2s0 inet manual

auto vmbr0
iface vmbr0 inet static
        address 172.16.150.202/21
        gateway 172.16.144.1
        bridge-ports enp2s0 
        bridge-stp off
        bridge-fd 0

auto vxlan100
iface vxlan100
  vxlan-id 100

auto vmbr100
iface vmbr100
  address 192.168.24.10/24
  bridge-ports vxlan100
  post-up bridge fdb add 00:00:00:00:00:00 dev vxlan100 dst 172.16.150.203
```

```bash
# Check configuration syntax.
$ sudo ifreload --syntax-check --all

# Check configuration syntax using verbose mode.
$ sudo ifreload --syntax-check --all --verbose

# Check configuration syntax using debug mode.
$ sudo ifreload --syntax-check --all --debug

# Perform a verbose dry run.
$ sudo ifreload --no-act --all --verbose

# Reload network configuration.
$ ifreload --all
```

Simple, but useful utility.

