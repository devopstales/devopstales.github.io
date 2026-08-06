---
title: Enable Proxmox PCIe Passthrough
url: https://devopstales.github.io/virtualization/proxmox-pci-passthrough/
date: 2022-03-08
keywords: Proxmox, Proxmox VE, PCIe Passthrough
---


Proxmox VE allows the passthrough of PCIe devices to individual virtual machines. In this blog post I will show you how you can configure it.
<!--more-->



### Grub Configuration

```bash
nano /etc/default/grub
# Example for an Intel system:
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

# Example for an AMD system:
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
```

Then update the grub: 

```bash
update-grub
```

### Add kernel modules
```bash
nano /etc/modules
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

Save the file and update the initramfs:

```bash
update-initramfs -u -k all
```

### Perform restart

```bash
init 6
```

### Check function

```bash
cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-5.4.128-1-pve root=/dev/mapper/pve-root ro quiet intel_iommu=on iommu=pt
```

```bash
dmesg |grep -e DMAR -e IOMMU -e AMD-Vi
[...]
[    0.273374] DMAR: IOMMU enabled
[...]
[    1.722014] DMAR: Intel(R) Virtualization Technology for Directed I/O
```

```bash
lsmod | grep vfio
vfio_pci               49152  0
vfio_virqfd            16384  1 vfio_pci
irqbypass              16384  2 vfio_pci,kvm
vfio_iommu_type1       32768  0
vfio                   32768  2 vfio_iommu_type1,vfio_pci
```

### Configuration Ethernet network card passthrough

Show PCI devices:

Identify network card:
```bash
ll /sys/class/net/eno1
lrwxrwxrwx 1 root root 0 Aug  6 12:09 /sys/class/net/eno1 -> ../../devices/pci0000:00/0000:00:01.1/0000:09:00.0/net/eno1
```

```bash
lspci | grep -iE --color 'network|ethernet'
05:00.0 Ethernet controller: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection (rev 01)
05:00.1 Ethernet controller: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection (rev 01)
09:00.0 Ethernet controller: Intel Corporation I350 Gigabit Network Connection (rev 01)
09:00.1 Ethernet controller: Intel Corporation I350 Gigabit Network Connection (rev 01)
```

### Add device to VM

```bash
nano /etc/pve/qemu-server/108.conf
...
hostpci0: 0000:09:00.0
# for multiple ones
hostpci0: 0000:09:00.0;0000:09:00.1
```

