---
title: Proxmox node removal
url: https://devopstales.github.io/linux/proxmox-node-remove/
date: 2019-03-07
keywords: proxmox ve, proxmox cluster
---


The correct way to remove nod from proxmox cluster.

<!--more-->

### Display all active nodes
```
root@proxmox-node2:~# pvecm nodes
Membership information
----------------------
Nodeid Votes Name
1 1 proxmox-node1 (local)
2 1 proxmox-node2
3 1 proxmox-node3
4 1 proxmox-node4
```

### Shutdown node and remove
```
root@proxmox-node2:~# pvecm delnode proxmox-node3

root@proxmox-node2:~# ls -l /etc/pve/nodes/
proxmox-node1 proxmox-node2 proxmox-node3 proxmox-node4

root@proxmox-node2:~# rm -rf /etc/pve/nodes/proxmox-node3
```

