---
title: Proxmox: Customize Web UI with dark theme
url: https://devopstales.github.io/linux/proxmox-dark-theme/
date: 2023-01-11
keywords: proxmox ve, proxmox cluster, proxmox ve, dark mode, dark theme
---


In thist post I will show you how to customize Proxmox VE Web UI with dark theme.

<!--more-->

There is no official option to set a dark mode on proxmox but there is a third party theme pack to customize your ui:

### Installation

The installation is done via the CLI utility. Run the following commands on the PVE node serving the Web UI: Clearing browser cache is necessary to see the changes.

```bash
wget https://raw.githubusercontent.com/Weilbyte/PVEDiscordDark/master/PVEDiscordDark.sh
```

Make the script executable and run it.

```bash
chmod +x PVEDiscordDark.sh 
sudo ./PVEDiscordDark.sh install
```

Installation process should complete in seconds.

```bash
✔ Backing up template file
✔ Downloading stylesheet
✔ Downloading patcher
✔ Applying changes to template file
✔ Downloading images (29/29)
Theme installed.
```

![Example image](/img/include/proxmox-darc01.webp)

### Update

The theme can be updated to new release with the command:

```bash
$ sudo ./PVEDiscordDark.sh update
✔ Removing stylesheet
✔ Removing patcher
✔ Reverting changes to template file
✔ Removing images
Theme uninstalled.
✔ Backing up template file
✔ Downloading stylesheet
✔ Downloading patcher
✔ Applying changes to template file
✔ Downloading images (29/29)
Theme installed.
```

### Uninstall

To uninstall the theme, simply run the utility with the uninstall command. Clearing browser cache is necessary to see the changes.

```bash
$ sudo ./PVEDiscordDark.sh uninstall
✔ Removing stylesheet
✔ Removing patcher
✔ Reverting changes to template file
✔ Removing images
Theme uninstalled.
```

Enjoy your new look on Proxmox VE GUI. 

