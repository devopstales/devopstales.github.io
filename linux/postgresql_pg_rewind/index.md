---
title: PostgreSQL: pg_rewind
url: https://devopstales.github.io/linux/postgresql_pg_rewind/
date: 2020-02-05
keywords: postgresql 12, pg_rewind, streaming replication
---


In this pos I will show you how to perform a rewind on a broken Streamin replication.

<!--more-->

### create test db
```
# master:
sudo -iu postgres psql -c "create database test1;"
```

### break the replication
```
# slaeve:
sudo -u postgres /usr/pgsql-12/bin/pg_ctl stop -m fast -D /var/lib/pgsql/12/data/
```

generate data in master
```
# master:
sudo -iu postgres /usr/pgsql-12/bin/pgbench -i -s 100 -d test1
```

### perform the rewind
```
# slaeve:
sudo -u postgres /usr/pgsql-12/bin/pg_rewind  --target-pgdata=/var/lib/pgsql/12/data/ --source-server="host=192.168.0.110 user=admin password=Password1 dbname=test1" -P
systemctl start postgresql-12
systemctl status postgresql-12
```

### test the replication status
```
/usr/pgsql-12/bin/pg_controldata -D /var/lib/pgsql/12/data/ | grep cluster
Database cluster state:               in archive recovery
```

```
# master:
sudo -u postgres psql -x -c "select * from pg_stat_replication"
could not change directory to "/root": Permission denied
-[ RECORD 1 ]----+------------------------------
pid              | 11548
usesysid         | 16384
usename          | replica_user
application_name | walreceiver
client_addr      | 192.168.0.111
client_hostname  |
client_port      | 48592
backend_start    | 2020-02-08 10:49:45.449591+01
backend_xmin     |
state            | streaming
sent_lsn         | 1/448638D0
write_lsn        | 1/448638D0
flush_lsn        | 1/448638D0
replay_lsn       | 1/448638D0
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2020-02-08 10:50:03.026266+01
```

