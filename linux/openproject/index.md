---
title: Install Openproject
url: https://devopstales.github.io/linux/openproject/
date: 2019-04-05
---


Openproject is a free and open source, web-based project management and issue tracking tool.

<!--more-->

### Install OpenProject
```
sudo wget -O /etc/yum.repos.d/openproject-ce.repo https://dl.packager.io/srv/opf/openproject-ce/stable/8/installer/el/7.repo

yum install openproject memcached -y
service memcached start
```

### Install Postgresql

In a previous post I wrote about how to [Install PostgreSQL 10]({{< relref "/linux/install-postgresql.md#install-postgresql-9-6-on-centos-7" >}})

```
su - postgres
createuser project
psql
ALTER USER project WITH ENCRYPTED password 'Password1';
CREATE DATABASE project WITH ENCODING='UTF8' OWNER=project;
\q
```

### Configure OpenProject
```
openproject configure
```

### Reset Password
```
openproject run console

admin = User.find_by(login: 'admin')
admin.password = 'Password11' # minimum 10 characters
admin.password_confirmation = 'Password11'

admin.save! # Watch the output for errors
```

