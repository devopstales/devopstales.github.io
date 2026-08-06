---
title: Replace CEPH SSD journal disk
url: https://devopstales.github.io/linux/ceph-change-journal-ssd/
date: 2021-12-12
---


In this post I will show you how you can change the end of life journal SSD in Ceph.
<!--more-->

### Replace a SSD disk used as journal for filestore

Let's suppose that we need to replace /dev/nvme0n1. This device is used for journal for osd.10 and osd.11:

```bash
[root@ceph-osd-02 ~]# ceph device ls | grep ceph-osd-02
ST6000NM0115-1YZ110_ZAD5KF07                    ceph-osd-02:sda                        osd.10
ST6000NM0115-1YZ110_ZAD5N8P7                    ceph-osd-02:sdb                        osd.11
Samsung_SSD_970_EVO_Plus_250GB_S4EUNJ0N111052K  ceph-osd-02:nvme0n1                    osd.10 osd.11
```

Let's tell ceph to not rebalance the cluster as we stop these OSDs for maintenance: 

```bash
[root@ceph-osd-02 ~]# ceph osd set noout
```

Let's stop the affected OSDs: 

```bash
[root@ceph-osd-02 ~]# systemctl stop ceph-osd@10.service
[root@ceph-osd-02 ~]# systemctl stop ceph-osd@11.service
```

Let's flush the journals for these OSDs: 

```bash
[root@ceph-osd-02 ~]# ceph-osd -i 10 --flush-journal
[root@ceph-osd-02 ~]# ceph-osd -i 11 --flush-journal
```

Backup nvme0n1 partition table.

```bash
[root@ceph-osd-02 ~]# sfdisk -l /dev/nvme0n1 > nvme0n1.partition.table.txt
```

Let's replace the device nvme0n1. In case, let's zap it: 

```bash
[root@ceph-osd-02 ~]# ceph-disk zap /dev/nvme0n1
```

Let's partition the new disk, using this script:

```bash
#!/bin/bash
 
osds="10 11"
journal_disk=/dev/nvme0n1
part_number=0
for osd_id in $osds; do
  part_number=$((part_number+1))
  journal_uuid=$(cat /var/lib/ceph/osd/ceph-$osd_id/journal_uuid)
  echo "journal_uuid: ${journal_uuid}"
  echo "part_number: ${part_number}"
  sgdisk --new=${part_number}:0:+30720M --change-name=${part_number}:'ceph journal' --partition-guid=${part_number}:$journal_uuid --typecode=${part_number}:45b0969e-9b03-4f30-b4c6-b4b80ceff106 --mbrtogpt -- $journal_disk
done
```

OR with backup:

```bash
[root@ceph-osd-02 ~]# sfdisk /dev/nvme0n1 < nvme0n1.partition.table.txt
```

Then:

```bash
[root@ceph-osd-02 ~]# ceph-osd --mkjournal -i 10
[root@ceph-osd-02 ~]# ceph-osd --mkjournal -i 11
```

Let's restart the osds: 

```bash
[root@ceph-osd-02 ~]# systemctl restart ceph-osd@10.service
[root@ceph-osd-02 ~]# systemctl restart ceph-osd@11.service
```

Finally: 

```bash
[root@ceph-osd-02 ~]# ceph osd unset noout
```

### Replace a SSD disk used as db for bluestore

Let's suppose that we need to replace /dev/nvme0n1. This device is used for journal for osd.10 and osd.11:

```bash
[root@ceph-osd-02 ~]# ceph device ls | grep ceph-osd-02
ST6000NM0115-1YZ110_ZAD5KF07                    ceph-osd-02:sda                        osd.10
ST6000NM0115-1YZ110_ZAD5N8P7                    ceph-osd-02:sdb                        osd.11
Samsung_SSD_970_EVO_Plus_250GB_S4EUNJ0N111052K  ceph-osd-02:nvme0n1                    osd.10 osd.11
```

Check the LVM paricioning on the nvme0n1: 

```bash
[root@ceph-osd-02 ~]# lsblk
sda                                                                                                     8:0    0   5.5T  0 disk
└─ceph--a2d09b40--caa4--4720--8953--5e86750da005-osd--block--de012ee4--60c4--4623--a98c--20b3256a6587 253:6    0   5.5T  0 lvm
sdb                                                                                                     8:16   0   5.5T  0 disk
└─ceph--01fefec3--2549--40dc--b03e--ea1cbf0c22f1-osd--block--df90bd50--cd26--4306--8c4a--6d97148870e8 253:8    0   5.5T  0 lvm
...
nvme0n1                                                                                               259:0    0 232.9G  0 disk
├─ceph--2dd99fb0--5e5a--4795--a14d--8fea42f9b4e9-osd--db--6463679d--ccd6--4988--a4fa--6bb0037b8f7a    253:5    0   115G  0 lvm
└─ceph--2dd99fb0--5e5a--4795--a14d--8fea42f9b4e9-osd--db--3b39c364--92cb--41c4--8150--ce7f4bdb4b2c    253:7    0   115G  0 lvm

vgdisplay -v
```

We find the volume groups used for db and block. This physical devices in our case it is: 
db: `ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9`
block1: `ceph-a2d09b40-caa4-4720-8953-5e86750da005`
block2: `ceph-01fefec3-2549-40dc-b03e-ea1cbf0c22f1`

```bash
[root@ceph-osd-02 ~]# vgdisplay -v ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9
  --- Volume group ---
  VG Name               ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  5
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               232.88 GiB
  PE Size               4.00 MiB
  Total PE              59618
  Alloc PE / Size       58880 / 230.00 GiB
  Free  PE / Size       738 / 2.88 GiB
  VG UUID               f29Vag-1PrI-fo7x-Dvhm-TNDl-2cfY-5hFY33

  --- Logical volume ---
  LV Path                /dev/ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9/osd-db-6463679d-ccd6-4988-a4fa-6bb0037b8f7a
  LV Name                osd-db-6463679d-ccd6-4988-a4fa-6bb0037b8f7a
  VG Name                ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9
  LV UUID                UWFXM5-5ZmF-Kb4f-jTqc-KuqZ-IWc7-UjHhXY
  LV Write Access        read/write
  LV Creation host, time ceph-osd-02, 2021-06-04 21:48:01 +0200
  LV Status              available
  # open                 12
  LV Size                115.00 GiB
  Current LE             29440
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:5

  --- Logical volume ---
  LV Path                /dev/ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9/osd-db-3b39c364-92cb-41c4-8150-ce7f4bdb4b2c
  LV Name                osd-db-3b39c364-92cb-41c4-8150-ce7f4bdb4b2c
  VG Name                ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9
  LV UUID                e52dYE-FuRK-TMPv-U8vx-38pv-KdfE-R0fwBo
  LV Write Access        read/write
  LV Creation host, time ceph-osd-02, 2021-06-04 21:48:30 +0200
  LV Status              available
  # open                 12
  LV Size                115.00 GiB
  Current LE             29440
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:7

  --- Physical volumes ---
  PV Name               /dev/nvme0n1
  PV UUID               4bVZmc-Vku7-rWPd-RHrn-xUFf-WrYb-yidijN
  PV Status             allocatable
  Total PE / Free PE    59618 / 738
```

 I.e. it is used as db for OSD 10-11

Let's 'disable these OSDs':

```bash
[root@c-osd-5 /]# ceph osd crush reweight osd.10 0
reweighted item id 10 name 'osd.10' to 0 in crush map
[root@c-osd-5 /]# ceph osd crush reweight osd.11 0
reweighted item id 11 name 'osd.11' to 0 in crush map
```

Wait that the status is HEALTH-OK. Then destroy the osds(be sure to have saved the mappings first !!):

```bash
[root@c-osd-5 /]# ll /var/lib/ceph/osd/ceph-10/ | grep block
lrwxrwxrwx 1 ceph ceph  93 Jun  4 21:48 block -> /dev/ceph-a2d09b40-caa4-4720-8953-5e86750da005/osd-block-de012ee4-60c4-4623-a98c-20b3256a6587
lrwxrwxrwx 1 ceph ceph  90 Jun  4 21:48 block.db -> /dev/ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9/osd-db-6463679d-ccd6-4988-a4fa-6bb0037b8f7a

[root@c-osd-5 /]# ll /var/lib/ceph/osd/ceph-11/ | grep block
lrwxrwxrwx 1 ceph ceph  93 Jun  4 21:48 block -> /dev/ceph-01fefec3-2549-40dc-b03e-ea1cbf0c22f1/osd-block-df90bd50-cd26-4306-8c4a-6d97148870e8
lrwxrwxrwx 1 ceph ceph  90 Jun  4 21:48 block.db -> /dev/ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9/osd-db-3b39c364-92cb-41c4-8150-ce7f4bdb4b2c
```

```bash
[root@ceph-osd-02 ~]# ceph osd out osd.10
[root@ceph-osd-02 ~]# ceph osd out osd.11

[root@ceph-osd-02 ~]# ceph osd crush remove osd.10
[root@ceph-osd-02 ~]# ceph osd crush remove osd.11

[root@ceph-osd-02 ~]# systemctl stop ceph-osd@10.service
[root@ceph-osd-02 ~]# systemctl stop ceph-osd@11.service

[root@ceph-osd-02 ~]# ceph auth del osd.10
[root@ceph-osd-02 ~]# ceph auth del osd.11

[root@ceph-osd-02 ~]# ceph osd rm osd.10
[root@ceph-osd-02 ~]# ceph osd rm osd.11

[root@ceph-osd-02 ~]# umount /var/lib/ceph/osd/ceph-10
[root@ceph-osd-02 ~]# umount /var/lib/ceph/osd/ceph-11
```

Destroy the volume group created on this SSD disk (be sure to have saved the vgdisplay output first !):

```bash
[root@ceph-osd-02 ~]# vgdisplay -v ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9 > ceph-vg.txt

[root@ceph-osd-02 ~]# vgremove ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9
```

Replace the SSD disk. Suppose the new one is always called vdk. Recreate volume group and logical volume (refer to the previous vgdisplay output):

```bash
[root@c-osd-5 /]# vgcreate ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9 /dev/vdk 
  Physical volume "/dev/vdk" successfully created.
  Volume group "ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9" successfully created
[root@c-osd-5 /]# lvcreate -L 115GB -n osd-db-6463679d-ccd6-4988-a4fa-6bb0037b8f7a ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9
  Logical volume "osd-db-6463679d-ccd6-4988-a4fa-6bb0037b8f7a" created.
[root@c-osd-5 /]# lvcreate -L 115GB -n osd-db-3b39c364-92cb-41c4-8150-ce7f4bdb4b2c ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9
[root@c-osd-5 /]# 
```

Let's do a lvm zap:

```bash
[root@c-osd-5 /]# ceph-volume lvm zap /var/lib/ceph/osd/ceph-10/block
[root@c-osd-5 /]# ceph-volume lvm zap /var/lib/ceph/osd/ceph-11/block
```

Let's create the OSD:

```bash
[root@c-osd-5 /]# ceph-volume lvm create --bluestore --data ceph-a2d09b40-caa4-4720-8953-5e86750da005/osd-block-de012ee4-60c4-4623-a98c-20b3256a6587 --block.db ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9/osd-db-6463679d-ccd6-4988-a4fa-6bb0037b8f7a
[root@c-osd-5 /]# ceph-volume lvm create --bluestore --data ceph-01fefec3-2549-40dc-b03e-ea1cbf0c22f1/osd-block-df90bd50-cd26-4306-8c4a-6d97148870e8 --block.db ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9/ceph-2dd99fb0-5e5a-4795-a14d-8fea42f9b4e9
```

