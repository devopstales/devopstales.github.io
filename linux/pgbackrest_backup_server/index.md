---
title: pgBackRest Backup server
url: https://devopstales.github.io/linux/pgbackrest_backup_server/
date: 2020-02-20
keywords: postgresql 12, postgresql 11, pgsql, backup, pgBackRest
---


In this article I will show you how to use pgBackRest in a dedicated backup server to backup remote PostgreSQL servers.

<!--more-->


### Installation

```
yum install epel-release -y
yum update -y
rpm -Uvh https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-7-x86_64/pgdg-centos10-10-2.noarch.rpm
yum install -y pgbackrest
```

### Allow passwordles ssh for hosts

Generate ssh key on all hosts and copy ssh keys
```
sudo -u postgres ssh-keygen -f /var/lib/pgsql/.ssh/id_rsa
sudo -u postgres restorecon -R /var/lib/pgsql/.ssh
```

```
sudo -u postgres ssh-copy-id -i /var/lib/pgsql/.ssh/id_rsa postgres@backup-server

sudo -u postgres ssh-copy-id -i /var/lib/pgsql/.ssh/id_rsa postgres@postgresql1
```

### Configure pgBackRest on postgresql12 server
```
nano /etc/pgbackrest.conf

[global]
repo1-host=backup-server
repo1-host-user=postgres
process-max=2
log-level-console=info
log-level-file=debug

[postgresql12]
pg1-path=/var/lib/pgsql/12/data
```

### Configure PostgreSQL on postgresql12 server
```
nano /var/lib/pgsql/12/data/postgresql.conf
...
archive_mode = on
archive_command = 'pgbackrest --stanza=postgresql1 archive-push %p'
...

systemctl restart postgresql12
```

### Configure pgBackRest on backup-server
```
nano /etc/pgbackrest.conf

[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=1
process-max=2
log-level-console=info
log-level-file=debug
start-fast=y
stop-auto=y

[postgresql12]
pg1-path=/var/lib/pgsql/12/data
pg1-host=postgresql1
pg1-host-user=postgres
```

### Perform a backup
Before we can start a backup we need to initialize the backup repository on backup-server.
```
sudo -iu postgres pgbackrest --stanza=postgresql12 stanza-create
```

Finally, check the configuration on all hosts.
```
sudo -iu postgres pgbackrest --stanza=postgresql12 check
```

```
sudo -iu postgres pgbackrest --stanza=postgresql12 --type=full backup
```

### Show backup information
```
$ sudo -iu postgres pgbackrest info
stanza: postgresql12
    status: ok
    cipher: none

    db (current)
        wal archive min/max (11-1): 000000010000000000000005/000000010000000000000005

        full backup: 20200219-091209F
            timestamp start/stop: 2020-02-19 09:12:09 / 2020-02-19 09:12:21
            wal start/stop: 000000010000000000000005 / 000000010000000000000005
            database size: 23.5MB, backup size: 23.5MB
            repository size: 2.8MB, repository backup size: 2.8MB
```

### Restore a backup
The restore command can then be used on the postgresql host.

```
sudo systemctl stop postgresql12

sudo -iu postgres pgbackrest --stanza=postgresql12 restore
```

Restore only test database:
```
sudo -iu postgres pgbackrest --stanza=postgresql12 --delta --db-include=test restore
```

