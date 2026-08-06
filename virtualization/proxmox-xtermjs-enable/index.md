---
title: How to: Enable Serial Console for guest virtual machine (VM) on Proxmox VE (PVE)
url: https://devopstales.github.io/virtualization/proxmox-xtermjs-enable/
date: 2022-04-19
keywords: proxmox ve, proxmox cluster
---


This article explains how to redirect messages to a serial console in on Debian and use Serial Console on Proxmox VE.
<!--more-->

We need to add serial0 to the virtual machine. Use following command to add the serial0 port

```bash
qm set [vmid] -serial0 socket
```

Now we need to configure the Debian:

```bash
nano /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="console=ttyS0,115200n8 console=tty1"
GRUB_CMDLINE_LINUX=""
GRUB_TERMINAL="serial console"
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
```

Now we need to configure the REHEL:

```bash
echo 'GRUB_CMDLINE_LINUX="quiet console=tty0 console=ttyS0,115200"' >> /tmp/grub

nano /etc/default/grub
```

Add `console=tty0 console=ttyS0,115200` to the end of the line `GRUB_CMDLINE_LINUX`

Now we need to update the grub by using following command

```bash
# Debian/Ubuntu etc.
update-grub
# RHEL/CentOS/Fedora
grub2-mkconfig --output=/boot/grub2/grub.cfg
```

Set to autostart the serial consol service.

```bash
mkdir -p /etc/systemd/system/serial-getty@ttyS0.service.d/
nano /etc/systemd/system/serial-getty@ttyS0.service.d/override.conf
[Service]
ExecStart=
ExecStart=-/sbin/agetty -o '-p -- \\u' 115200 %I $TERM

systemctl daemon-reload
systemctl restart serial-getty@ttyS0.service
systemctl enable serial-getty@ttyS0.service
```

Reboot and test the autostart of the Serial Console

```bash
init 6
ps -ef | grep ttyS0
systemctl status serial-getty@ttyS0.service
```

