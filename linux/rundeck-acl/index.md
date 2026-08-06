---
title: Configure Rundeck ACL
url: https://devopstales.github.io/linux/rundeck-acl/
date: 2019-12-10
---


In this post I will configure access control in Rundeck.

<!--more-->

### Configurate AD groups in rundeck
The modification of the `web.xml` no longer needed after 3.0.x.
```
nano /var/lib/rundeck/exp/webapp/WEB-INF/web.xml
        <security-role>
               <role-name>rundeck-administrators</role-name>
               <role-name>rundeck-project</role-name>
        </security-role>
```

### Configure the privilege for AD group
```
nano /etc/rundec/admin.aclpolicy

description: Admin, all access.
context:
  project: '.*' # all projects
for:
  resource:
    - allow: '*' # allow read/create all kinds
  adhoc:
    - allow: '*' # allow read/running/killing adhoc jobs
  job:
    - allow: '*' # allow read/write/delete/run/kill of all jobs
  node:
    - allow: '*' # allow read/run for all nodes
by:
  group: rundeck-administrators

---

description: Admin, all access.
context:
  application: 'rundeck'
for:
  resource:
    - allow: '*' # allow create of projects
  project:
    - allow: '*' # allow view/admin of all projects
  project_acl:
    - allow: '*' # allow admin of all project-level ACL policies
  storage:
    - allow: '*' # allow read/create/update/delete for all /keys/* storage content
by:
  group: rundeck-administrators
---

description: rundeck-project  PROJECT all access.
context:
  project: 'PROJECT'
for:
  resource:
    - allow: '*' # allow read/create all kinds
  adhoc:
    - allow: '*' # allow read/running/killing adhoc jobs
  job:
    - allow: '*' # allow read/write/delete/run/kill of all jobs
  node:
    - allow: '*' # allow read/run for all nodes
by:
  group: rundeck-project

---

description: rundeck-project, all access.
context:
  application: 'rundeck'
for:
  project:
    - match:
        name: 'PROJECT'
      allow: [read]
  system:
    - match:
        name: '.*'
      allow: [read]
  storage:
    - equals:
        path: 'keys'
      allow: [read]
    - match:
        path: 'keys/id_rsa*'
      allow: [read]
by:
  group: rundeck-project
```

