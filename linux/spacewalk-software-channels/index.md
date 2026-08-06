---
title: Configure spacewalk 2.9
url: https://devopstales.github.io/linux/spacewalk-software-channels/
date: 2019-04-29
---


In this post I will show you how to create software channels on spacewalk server.
<!--more-->
Login to the spacewalk web interface

### Get GPG key
First, we need have an extracted GPG key information, To get the key information download it and extract it using GPG command.


```
cd /etc/pki/rpm-gpg
wget http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-7
gpg --with-fingerprint RPM-GPG-KEY-CentOS-7
wget https://copr-be.cloud.fedoraproject.org/results/@spacewalkproject/spacewalk-2.9/pubkey.gpg -o RPM-GPG-KEY-spacewalk-2.9
gpg --with-fingerprint RPM-GPG-KEY-spacewalk-2.9

cp /etc/pki/rpm-gpg/RPM-GPG-KEY-* /var/www/html/pub/
```

### Create software chanel
Channels (top) > Manage Software Channels(Left side pane) > Create Channel(Right side top corner).<br>

![Example image](/img/include/spacewalk_1.webp)


|                 |                        |
|-----------------|------------------------|
| Channel Name    | centos 7 base - x86_64 |
| Channel Label   | centos7-base-x86_64    |
| Parent Channel  | none                   |
| Architecture    | x86_64                 |
| Channel Summary | centos7-base-x86_64    |
| GPG Key URL     | file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7 |
| GPG Key ID      | F4A80EB5 |
| GPG Fingerprint | 6341 AB27 53D7 8A78 A7C2 7BB1 24C6 A8A7 F4A8 0EB5 |

![Example image](/img/include/spacewalk_2.webp)

### Create repositories

Channels (Top) > Manage Software Channels (Left side pane) > Manage Repositories (Left side pane) > Create Repository(Right side top corner).

| Repository Label: | centos7_base_x86_64_repo |
|-------------------|--------------------------|
| Repository URL: | http://mirror.centos.org/centos/7/os/x86_64/ |
| Repository Type: | yum |

| Repository Label: | centos7_update_x86_64_repo |
|-------------------|--------------------------|
| Repository URL: | http://mirror.centos.org/centos/7/updates/x86_64/ |
| Repository Type: | yum |

| Repository Label: | centos7_extra_x86_64_repo |
|-------------------|--------------------------|
| Repository URL: | http://mirror.centos.org/centos/7/extras/x86_64/ |
| Repository Type: | yum |

| Repository Label: | centos7_opnstk_x86_64_repo |
|-------------------|--------------------------|
| Repository URL: | http://mirror.centos.org/centos/7/cloud/x86_64/ |
| Repository Type: | yum |

### Adding Repository to Channel
Channels (Top) > Manage Software Channels (Left side pane) > Centos 7 Base x86_64  > Repositories (Tab)  > centos7_base_x86_64_repo (Check box)  > Update Repositories (Bottom right corner). <br>

![Example image](/img/include/spacewalk_3.webp)

### Creating activation Key
System  > ( Top menu) Activation Keys (Left Side pane)  > Create Key (Right side top corner) > Fill description <br>

|   |   |
|---|---|
| Description: | CentOS Linux 7 x86_64 |
| Key: | centoslinux7-x86_64 |
| Usage: |  |
| Base channels: | Centos 7 Base - x86_64 |
| Add-On Entitlements: | Choose all available feature you about to use. |

![Example image](/img/include/spacewalk_4.webp)
![Example image](/img/include/spacewalk_5.webp)

### Start Syncing repositories
```
spacewalk-repo-sync --channel centos-7-base-x86_64 --type yum
tail -f /var/log/rhn/reposync/centos-7-base-x86_64.log

df -hP /var/satellite
```

