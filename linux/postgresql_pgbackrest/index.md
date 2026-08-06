---
title: PostgreSQL backup with pgBackRest
url: https://devopstales.github.io/linux/postgresql_pgbackrest/
date: 2020-02-19
keywords: postgresql 12, postgresql 11, pgsql, backup, pgBackRest
---


In this article I will show you how to use pgBackRest as a backup program for PostgreSQL.

<!--more-->

pgBackRest aims to be a simple, reliable backup and restore solution that can seamlessly scale up to the largest databases and workloads by utilizing algorithms that are optimized for database-specific requirements.

> A backup is a consistent copy of a database cluster that can be restored to recover from a hardware failure, to perform Point-In-Time Recovery, or to bring up a new standby. For individual database backups, a tool such as `pg_dump` must be used.

### Installation

```
yum install epel-release -y
yum update -y
rpm -Uvh https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-7-x86_64/pgdg-centos10-10-2.noarch.rpm
yum install -y pgbackrest
```

### Configure pgBackRest
```
cp /etc/pgbackrest.conf /etc/pgbackrest.conf.bck

nano /etc/pgbackrest.conf
[global]
repo1-path=/var/lib/pgsql/12/backups
log-level-console=info
log-level-file=debug
start-fast=y

[backup_stanza]
pg1-path=/var/lib/pgsql/12/data
repo1-retention-full=1
```

The configuration is based on two section the `global` what is specify the repository where you stores the backups and WAL segments archives and a `stanza`, in my case `backup_stanza`.

A stanza defines the backup configuration for a specific PostgreSQL database cluster. The stanza section must define the database cluster path and host/user if the database cluster is remote. Also, any global configuration sections can be overridden to define stanza-specific settings.

### Configure PostgreSQL
```
nano /var/lib/pgsql/12/data/postgresql.conf
...
archive_mode = on
archive_command = 'pgbackrest --stanza=backup_stanza archive-push %p'
...

systemctl restart postgresql12
```

### Perform a backup
Before we can start a backup we need to initialize the backup repository.
```
$ sudo -iu postgres pgbackrest --stanza=backup_stanza stanza-create
P00   INFO: stanza-create command begin 2.23: ...
P00   INFO: stanza-create command end: completed successfully

$ sudo -iu postgres pgbackrest --stanza=backup_stanza check
P00   INFO: check command begin 2.23: ...
P00   INFO: WAL segment ... successfully stored in the archive at ...
P00   INFO: check command end: completed successfully
```

```
$ sudo -iu postgres pgbackrest --stanza=backup_stanza --type=full backup
P00   INFO: backup command begin 2.23: ...
P00   INFO: execute non-exclusive pg_start_backup() with label
        "pgBackRest backup started at ...": backup begins after the requested immediate checkpoint completes
P00   INFO: backup start archive = 000000010000000000000005, lsn = 0/5000028
P00   INFO: full backup size = 23.5MB
P00   INFO: execute non-exclusive pg_stop_backup() and wait for all WAL segments to archive
P00   INFO: backup stop archive = 000000010000000000000005, lsn = 0/5000130
P00   INFO: new backup label = 20200219-091209F
P00   INFO: backup command end: completed successfully
P00   INFO: expire command begin
P00   INFO: expire full backup 20200219-090152F
P00   INFO: remove expired backup 20200219-090152F
P00   INFO: expire command end: completed successfully
```

### Show backup information
```
$ sudo -iu postgres pgbackrest info
stanza: backup_stanza
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
```
sudo systemctl stop postgresql12

sudo -iu postgres pgbackrest --stanza=backup_stanza restore
P00   INFO: restore command begin 2.15: ...
P00   INFO: restore backup set 20200219-091209F
P00   INFO: write /var/lib/pgsql/11/data/recovery.conf
P00   INFO: restore global/pg_control (performed last to ensure aborted restores cannot be started)
P00   INFO: restore command end: completed successfully
```

Restore only test database:
```
# restor to the lastert backup
sudo -iu postgres pgbackrest --stanza=backup_stanza --delta --db-include=test restore
# restore to a specific verion
sudo -Hiu postgres pgbackrest --stanza=backup_stanza --delta --db-include=test --set=20200219-091209F restore
```

