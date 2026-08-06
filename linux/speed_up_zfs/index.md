---
title: How to speed up zfs resilver?
url: https://devopstales.github.io/linux/speed_up_zfs/
date: 2020-01-12
keywords: zfs on linux
---


In this article I will show you how speed up zfs.

<!--more-->

First we need to understand there is two type of zfs the FreeBSD/Solaris based and Linux based cald Zfs On Linux or ZOL.

When a device is replaced, a resilvering operation is initiated to move data from the good copies to the new device. This action is a form of disk scrubbing. RAIDz resilvering is very slow in OpenZFS-based zpools. Basically, it starts with every transaction that’s ever happened in the pool and plays them back one-by-one to the new drive. This is very IO-intensive. Some say if you’re using hard drives larger than 1TB and you are using OpenZFS, use mirror, not RAIDz* or use only SSD for RAIDz*.

In ZOL 0.8.0 they changed to a new resilvering solution. The previous resilvering algorithm repairs blocks from oldest to newest, which can degrade into a lot of small random I/O. The new resilvering algorithm uses a two-step process to sort and resilver blocks in LBA order. If you have an older version of ZOL or want even better performance you can tweak with the zfs configuration.


### Speed up resilvering
```
echo 0 > /sys/module/zfs/parameters/zfs_resilver_delay
echo 0 > /sys/module/zfs/parameters/zfs_scrub_delay
echo 512 > /sys/module/zfs/parameters/zfs_top_maxinflight
echo 8192 > /sys/module/zfs/parameters/zfs_top_maxinflight
echo 8000 > /sys/module/zfs/parameters/zfs_resilver_min_time_ms
```

### Speed up on pool
```
zpool set autoreplace=on zpool

echo 0 > /sys/module/zfs/parameters/zfs_scan_idle
echo 24 > /sys/module/zfs/parameters/zfs_vdev_scrub_min_active
echo 64 > /sys/module/zfs/parameters/zfs_vdev_scrub_max_active
echo 64 > /sys/module/zfs/parameters/zfs_vdev_sync_write_min_active
echo 32 > /sys/module/zfs/parameters/zfs_vdev_sync_write_max_active
echo 8 > /sys/module/zfs/parameters/zfs_vdev_sync_read_min_active
echo 32 > /sys/module/zfs/parameters/zfs_vdev_sync_read_max_active
echo 8 > /sys/module/zfs/parameters/zfs_vdev_async_read_min_active
echo 32 > /sys/module/zfs/parameters/zfs_vdev_async_read_max_active
echo 8 > /sys/module/zfs/parameters/zfs_vdev_async_write_min_active
```

