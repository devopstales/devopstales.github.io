---
title: Install CEHP Radosgateway on Proxmox
url: https://devopstales.github.io/virtualization/proxmox-ceph-radosgw/
date: 2019-06-14
---


RADOS Gateway  is an object storage interface in Ceph. It provides interfaces compatible with OpenStack Swift and Amazon S3.

<!--more-->

First create a keyring than generated the keys and added them to the keyring:
```bash
root@pve1:~# ceph-authtool --create-keyring /etc/ceph/ceph.client.radosgw.keyring

root@pve1:~# ceph-authtool /etc/ceph/ceph.client.radosgw.keyring -n client.radosgw.pve1 --gen-key
root@pve1:~# ceph-authtool /etc/ceph/ceph.client.radosgw.keyring -n client.radosgw.pve2 --gen-key
root@pve1:~# ceph-authtool /etc/ceph/ceph.client.radosgw.keyring -n client.radosgw.pve3 --gen-key
```

And then I added the proper capabilities and add the keys to the cluster:
```bash
root@pve1:~# ceph-authtool -n client.radosgw.pve1 --cap osd 'allow rwx' --cap mon 'allow rwx' /etc/ceph/ceph.client.radosgw.keyring
root@pve1:~# ceph-authtool -n client.radosgw.pve2 --cap osd 'allow rwx' --cap mon 'allow rwx' /etc/ceph/ceph.client.radosgw.keyring
root@pve1:~# ceph-authtool -n client.radosgw.pve3 --cap osd 'allow rwx' --cap mon 'allow rwx' /etc/ceph/ceph.client.radosgw.keyring
```

```bash
scp /etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.radosgw.keyring pve2:/etc/ceph/
scp /etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.radosgw.keyring pve3:/etc/ceph/
```

```bash
root@pve1:~# ceph -k /etc/ceph/ceph.client.admin.keyring auth add client.radosgw.pve1 -i /etc/ceph/ceph.client.radosgw.keyring
root@pve1:~# ceph -k /etc/ceph/ceph.client.admin.keyring auth add client.radosgw.pve2 -i /etc/ceph/ceph.client.radosgw.keyring
root@pve1:~# ceph -k /etc/ceph/ceph.client.admin.keyring auth add client.radosgw.pve3 -i /etc/ceph/ceph.client.radosgw.keyring
```

If you get the fallofing error: `handle_auth_bad_method server allowed_methods [2] but i only support [2]`

You Have a problem with your `/etc/ceph/ceph.client.admin.keyring` file:

```bash
sudo ceph --cluster ceph auth get-key client.admin
AQDxnppkhI2ZOBAAJ1VFYV6FvRi8vZyuUYzwZQ==

nano /etc/ceph/ceph.client.admin.keyring
[client.admin]
	key = AQDxnppkhI2ZOBAAJ1VFYV6FvRi8vZyuUYzwZQ==
	caps mds = "allow *"
	caps mgr = "allow *"
	caps mon = "allow *"
	caps osd = "allow *"
```

Copy to the other nodes:

```bash
scp /etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.radosgw.keyring pve2:/etc/ceph/
scp /etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.radosgw.keyring pve3:/etc/ceph/
```

Copy the rings to the proxmox ClusterFS
```bash
root@pve1:~# cp /etc/ceph/ceph.client.radosgw.keyring /etc/pve/priv
```

Add the following lines to `/etc/ceph/ceph.conf`:
```bash
[client.radosgw.pve1]
        host = pve1
        keyring = /etc/pve/priv/ceph.client.radosgw.keyring
        log file = /var/log/ceph/client.radosgw.$host.log
        rgw_dns_name = s3.devopstales.intra
        rgw_frontends = civetweb port=10.83.110.1:7480

[client.radosgw.pve2]
        host = pve2
        keyring = /etc/pve/priv/ceph.client.radosgw.keyring
        log file = /var/log/ceph/client.radosgw.$host.log
        rgw_dns_name = s3.devopstales.intra
        rgw_frontends = civetweb port=10.83.110.2:7480

[client.radosgw.pve3]
        host = pve3
        keyring = /etc/pve/priv/ceph.client.radosgw.keyring
        log file = /var/log/ceph/client.rados.$host.log
        rgw_dns_name = s3.devopstales.intra
        rgw_frontends = civetweb port=10.83.110.3:7480
```

Install the pcakages and start the service. If all goes well, RADOSGW will create some default pools for you.
```bash
root@pve1:~# apt install radosgw
root@pve1:~# service radosgw start

root@pve1:~# tail -f /var/log/ceph/client.rados.pve1.log
```

```bash
root@pve1:~# ceph osd pool application enable .rgw.root rgw
root@pve1:~# ceph osd pool application enable default.rgw.control rgw

root@pve1:~# ceph osd pool application enable default.rgw.data.root rgw
root@pve1:~# ceph osd pool application enable default.rgw.gc rgw
root@pve1:~# ceph osd pool application enable default.rgw.log rgw
root@pve1:~# ceph osd pool application enable default.rgw.users.uid rgw
root@pve1:~# ceph osd pool application enable default.rgw.users.email rgw
root@pve1:~# ceph osd pool application enable default.rgw.users.keys rgw
root@pve1:~# ceph osd pool application enable default.rgw.buckets.index rgw
root@pve1:~# ceph osd pool application enable default.rgw.buckets.data rgw
root@pve1:~# ceph osd pool application enable default.rgw.lc rgw
```

```bash
root@pve1:~#  ssh pve2 'apt install radosgw && service radosgw start'
root@pve1:~#  ssh pve3 'apt install radosgw && service radosgw start'

root@pve1:~#  ceph osd pool ls
```

```bash
root@pve1:~# radosgw-admin user create --uid=devopstales --display-name="devopstales" --email=devopstales@devopstales.intra
root@pve1:~# radosgw-admin user info devopstales

root@pve1:~# ceph osd pool application enable default.rgw.buckets.index rgw
root@pve1:~# ceph osd pool application enable default.rgw.buckets.data rgw

#for minio cli to create bucketceph osd pool create default.rgw.buckets.data 32
ceph osd pool create default.rgw.buckets.index 8
ceph osd pool set default.rgw.buckets.index pgp_num 8
ceph osd pool set default.rgw.buckets.index size 3
ceph osd pool application enable default.rgw.buckets.index rgw
 
ceph osd pool create default.rgw.buckets.data 32
ceph osd pool set default.rgw.buckets.data pgp_num 32
ceph osd pool set default.rgw.buckets.data size 3
ceph osd pool application enable default.rgw.buckets.data rgw
```

```bash
root@pve1:~# apt-get install s3cmd
root@pve1:~# s3cmd --configure
Access Key: xxxxxxxxxxxxxxxxxxxxxx
Secret Key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

root@pve1:~#  s3cmd mb s3://devopstales
Bucket 's3://devopstales/' created
```

