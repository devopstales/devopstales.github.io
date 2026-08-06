---
title: How to backup Graylog logs from elasticsearch
url: https://devopstales.github.io/linux/elasticsearch-backup/
date: 2019-09-07
keywords: graylog, elasticsearch
---


Graylog store the log data in elasticsearch so I will show you how to create and restore snapshot with elasticsearch.
<!--more-->

### Requirement

* elasticsearch 7.5

First you will need to add the repo.path location to your elasticsearch.yml. This is the local path of the folder where the snapshot files will store.

```
mkdir -p /mnt/elasticsearch-backup
chown -R elasticsearch. /mnt/elasticsearch-backup

cat >> /etc/elasticsearch/elasticsearch.yml << EOF
path.repo: ["/mnt/elasticsearch-backup"]
EOF

systemctl restart elasticsearch
```


### Elasticsearch
Elasticsearch needs to know the backup path by registering a backup repository:

```
curl -XPUT 'http://localhost:9200/_snapshot/my_backup' -d '{
  "type": "fs",
  "settings": {
     "location": "/mnt/elasticsearch-backup",
     "compress": true
  }
}'
```

### Create Backup
```
curl -XPUT "localhost:9200/_snapshot/my_backup/snapshot_1?wait_for_completion=true"

# list snapshots:
curl -XGET 'localhost:9200/_snapshot/my_backup/_all?pretty'
```

### Restore backup
```
curl -XPOST "localhost:9200/_snapshot/my_backup/snapshot_1/_restore?wait_for_completion=true"
```

### Delete snapshot
```
curl -XDELETE 'localhost:9200/_snapshot/my_backup/snapshot_1'
```

