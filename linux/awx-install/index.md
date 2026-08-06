---
title: Install AWX
url: https://devopstales.github.io/linux/awx-install/
date: 2019-04-30
keywords: ansible awx, ansible tower, configuration management
---


AWX is an open source web application that provides a user interface, REST API, and task engine for Ansible.
<!--more-->

### Install Postgresql
```
yum install epel-release -y
yum update -y
rpm -Uvh https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-7-x86_64/pgdg-centos10-10-2.noarch.rpm

yum -y install postgresql10-server postgresql10-contrib postgresql10
/usr/pgsql-10/bin/postgresql-10-setup initdb

sudo systemctl start postgresql-10
sudo systemctl enable postgresql-10

sudo -u postgres createuser -S awx
sudo -u postgres createdb -O awx awx
```


### Install requirements and AWX
```
# if selinux enabled
yum -y install policycoreutils-python
setsebool -P httpd_can_network_connect 1

# if firewall enabled
firewall-cmd --permanent --add-service=http
firewall-cmd --reload


yum -y install memcached ansible epel-release nginx
systemctl enable memcached
systemctl start memcached

yum install https://github.com/rabbitmq/erlang-rpm/releases/download/v20.1.7.1/erlang-20.1.7.1-1.el7.centos.x86_64.rpm
yum -y install https://dl.bintray.com/rabbitmq/all/rabbitmq-server/3.7.5/rabbitmq-server-3.7.5-1.el7.noarch.rpm
systemctl enable rabbitmq-server
systemctl start rabbitmq-server

cp -p /etc/nginx/nginx.conf{,.org}
wget -O /etc/nginx/nginx.conf https://raw.githubusercontent.com/sunilsankar/awx-build/master/nginx.conf
systemctl start nginx
systemctl enable nginx

yum -y install centos-release-scl centos-release-scl-rh
yum -y install rh-python36 rh-python36-Django rh-python36-django-split-settings \
rh-python36-django-qsstats-magic rh-python36-ansiconv rh-python36-prometheus_client \
rh-python36-python-memcached rh-python36-asn1crypto rh-python36-asgiref \
rh-python36-hyperlink rh-python36-Automat rh-python36-asgi_amqp rh-python36-uwsgi \
rh-python36-msgpack-python rh-python36-msgpack-python-debuginfo rh-python36-jsonpickle \
rh-python36-django-radius rh-python36-python-django-radius rh-python36-python-radius \
rh-python36-future rh-python36-pyrad rh-python36-netaddr rh-python36-tacacs_plus \
rh-python36-python3-saml rh-python36-xmlsec rh-python36-lxml rh-python36-defusedxml \
rh-python36-boto

wget -O /etc/yum.repos.d/ansible-awx.repo https://copr.fedorainfracloud.org/coprs/mrmeee/ansible-awx/repo/epel-7/mrmeee-ansible-awx-epel-7.repo
yum install -y ansible-awx


sudo -u awx scl enable rh-python36 rh-postgresql10 "awx-manage migrate"
echo "from django.contrib.auth.models import User; User.objects.create_superuser('admin', 'root@localhost', 'Password1')" \
| sudo -u awx scl enable rh-python36 rh-postgresql10 "awx-manage shell"

sudo -u awx scl enable rh-python36 rh-postgresql10 "awx-manage create_preload_data"
sudo -u awx scl enable rh-python36 rh-postgresql10 "awx-manage provision_instance --hostname=$(hostname)"
sudo -u awx scl enable rh-python36 rh-postgresql10 "awx-manage register_queue --queuename=tower --hostnames=$(hostname)"
```

### Configure database
```
systemctl start awx-cbreceiver
systemctl start awx-dispatcher
systemctl start awx-channels-worker
systemctl start awx-daphne
systemctl start awx-web

systemctl status awx-cbreceiver
systemctl status awx-dispatcher
systemctl status awx-channels-worker
systemctl status awx-daphne
systemctl status awx-web

systemctl enable awx-cbreceiver
systemctl enable awx-dispatcher
systemctl enable awx-channels-worker
systemctl enable awx-daphne
systemctl enable awx-web
```

