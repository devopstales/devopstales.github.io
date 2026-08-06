---
title: Opeshift elasticsearch search-guard error
url: https://devopstales.github.io/kubernetes/openshift-elasticsearch-error/
date: 2021-12-15
keywords: ansible, openshift 3.11, openshift okd, openshift container platform, rad hat openshift
---



In this post I will show You How you can fix elasticsearch search-guard index error.

<!--more-->

{{< content "/filedir/openshift.html" >}}

If you get the following error:

```bash
[2021-12-15 09:10:17,949][INFO ][container.run            ] Seeding the searchguard ACL index.  Will wait up to 604800 seconds.
[2021-12-15 09:10:18,027][INFO ][container.run            ] Seeding the searchguard ACL index.  Will wait up to 604800 seconds.
/etc/elasticsearch ~
Search Guard Admin v5
Will connect to localhost:9300 ... done
ERROR StatusLogger No Log4j 2 configuration file found. Using default configuration (logging only errors to the console), or user programmatically provided configurations. Set system property 'log4j2.debug' to show Log4j 2 internal initialization logging. See https://logging.apache.org/log4j/2.x/manual/configuration.html for instructions on how to configure Log4j 2
Elasticsearch Version: 5.6.13
Search Guard Version: <unknown>
Contacting elasticsearch cluster 'elasticsearch' ...
Clustername: logging-es
Clusterstate: RED
Number of nodes: 1
Number of data nodes: 1
```

Try to rerun the inicialization script:

```bash
oc get pods -l component=es
NAME                                      READY     STATUS    RESTARTS   AGE
logging-es-data-master-9fgtlhi4-3-d48rs   2/2       Running   0          21m

oc exec -c elasticsearch logging-es-data-master-9fgtlhi4-3-d48rs -- es_seed_acl
```

If you get the same log we need to delete the searchguard index and reinicilaize:

```bash
oc exec -c elasticearch logging-es-data-master-9fgtlhi4-3-d48rs --es_util --query=.searchguard -XDELETE
{"acknowledged":true}

oc exec -c elasticsearch logging-es-data-master-9fgtlhi4-3-d48rs -- es_seed_acl
[2021-12-15 09:15:47,762][INFO ][container.run            ] Seeding the searchguard ACL index.  Will wait up to 604800 seconds.
[2021-12-15 09:15:47,931][INFO ][container.run            ] Seeding the searchguard ACL index.  Will wait up to 604800 seconds.
/etc/elasticsearch ~
Search Guard Admin v5
Will connect to localhost:9300 ... done
ERROR StatusLogger No Log4j 2 configuration file found. Using default configuration (logging only errors to the console), or user programmatically provided configurations. Set system property 'log4j2.debug' to show Log4j 2 internal initialization logging. See https://logging.apache.org/log4j/2.x/manual/configuration.html for instructions on how to configure Log4j 2
Elasticsearch Version: 5.6.16
Search Guard Version: <unknown>
Contacting elasticsearch cluster 'elasticsearch' ...
Clustername: logging-es
Clusterstate: RED
Number of nodes: 1
Number of data nodes: 1
.searchguard index does not exists, attempt to create it ...
Populate config from /opt/app-root/src/sgconfig/
Will update 'config' with /opt/app-root/src/sgconfig/sg_config.yml
   SUCC: Configuration for 'config' created or updated
Will update 'roles' with /opt/app-root/src/sgconfig/sg_roles.yml
   SUCC: Configuration for 'roles' created or updated
Will update 'rolesmapping' with /opt/app-root/src/sgconfig/sg_roles_mapping.yml
   SUCC: Configuration for 'rolesmapping' created or updated
Will update 'internalusers' with /opt/app-root/src/sgconfig/sg_internal_users.yml
   SUCC: Configuration for 'internalusers' created or updated
Will update 'actiongroups' with /opt/app-root/src/sgconfig/sg_action_groups.yml
   SUCC: Configuration for 'actiongroups' created or updated
Done with success
```

