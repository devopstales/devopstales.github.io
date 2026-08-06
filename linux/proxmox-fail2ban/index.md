---
title: Proxmox: Potect your server with fail2ban
url: https://devopstales.github.io/linux/proxmox-fail2ban/
date: 2023-01-11
keywords: proxmox ve, proxmox cluster, fail2ban
---


In thist post I will show you how you can protect your Proxmox server from broutforce http and ssh login atacks with fail2ban.

<!--more-->

Out of the box Proxmox does not have any Brute Force protection si I decided to configure fail2ban to protect my home server.

On proxmox fail2ban is really easy to install:

```bash
apt-get install fail2ban
```

Now we have to create a configfile for the filter. For this we create the file `/etc/fail2ban/filter.d/proxmox.conf` and add the following content:

```bash
nano /etc/fail2ban/filter.d/proxmox.conf
[definition]
failregex = pvedaemon\[.*authentication failure; rhost=<HOST> user=.* msg=.*
ignoreregex =.
```

### Protecting the web interface

Add the following string to the end of this file `/etc/fail2ban/jail.local`: 

```bash
[proxmox]
enabled = true
port = https,http,8006
filter = proxmox
logpath = /var/log/daemon.log
maxretry = 3
# 1 hour
bantime = 3600
```

### Protecting ssh

Add the following string to the end of this file `/etc/fail2ban/jail.local`:

```bash
[sshd]
port = ssh
logpath = %(sshd_log)s
enabled = true
```

You can test your configuration trying to GUI login with a wrong password or user, and then issue the command:

```bash
fail2ban-regex /var/log/daemon.log /etc/fail2ban/filter.d/proxmox.conf
``` 

Once done we need to restart fail2ban

```bash
systemctl restart fail2ban

tail -f /var/log/fail2ban.log
2023-01-08 17:45:32,928 fail2ban.jail [49691]: INFO Jail 'sshd' started
2023-01-08 17:45:32,928 fail2ban.jail [49691]: INFO Jail 'proxmox' started
```

### Test

If you want to test the configuration once, you can do it easily via the following command.

```bash
fail2ban-regex /var/log/daemon.log /etc/fail2ban/filter.d/proxmox.conf

Running tests
=============

Use failregex filter file : proxmox, basedir: /etc/fail2ban
Use log file : /var/log/daemon.log
Use encoding : UTF-8

Results
=======

Failregex: 0 total

Ignoreregex: 0 total

Date template hits:
|- [# of hits] date format
| [370] {^LN-BEG}(?:DAY )?MON Day %k:Minute:Second(?:\.Microseconds)?(?: ExYear)?
`-

Lines: 370 lines, 0 ignored, 0 matched, 370 missed
[processed in 0.01 sec]

Missed line(s): too many to print.  Use --print-all-missed to print all 370 lines
```

If you want to see if your ban is working take a look at:

```bash
fail2ban-client status sshd
#or
fail2ban-client status proxmox
```

