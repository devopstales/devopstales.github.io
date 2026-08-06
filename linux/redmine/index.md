---
title: Install Redmine
url: https://devopstales.github.io/linux/redmine/
date: 2019-04-06
---


Redmine is a free and open source, web-based project management and issue tracking tool. I will install it on Ubuntu becous on CetOS there in no pre build package for redmine.

<!--more-->

### Install and configure Postgresql

In a previous post I wrote about how to [Install PostgreSQL 10]({{< relref "/linux/install-postgresql.md#install-postgresql-10-on-debian" >}})

```
su - postgres
createuser redmine
psql
ALTER USER redmine WITH ENCRYPTED password 'Password1';
CREATE DATABASE redmine WITH ENCODING='UTF8' OWNER=redmine;
\q
```

### Install Redmine
```
apt install git redmine redmine-pgsql
chmod 777 /usr/share/redmine/instances/default/tmp/cache/
cd /usr/share/redmine
ruby bin/rails server webrick –e production
```

```
echo '[Unit]
Description=Redmine server
After=syslog.target
After=network.target

[Service]
Type=simple
User=redmine
Group=redmine
WorkingDirectory=/usr/share/redmine
ExecStart=/usr/bin/ruby /usr/share/redmine/bin/rails server webrick –e production

# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec=300

[Install]
WantedBy=multi-user.target' > /etc/systemd/system/redmine.service
```

```
apt install apache2 libapache2-mod-passenger
cp /usr/share/doc/redmine/examples/apache2-passenger-host.conf /etc/apache2/sites-available/redmine.conf
nano /etc/apache2/sites-available/redmine.conf

a2enmod passenger
a2enmod proxy
a2enmod rewrite
a2ensite redmine.conf
a2dissite 000-default
service apache2 reload
```

### Install Them
```
cd /usr/share/redmine/public/themes
# https://github.com/akabekobeko/redmine-theme-minimalflat2/releases
wget https://github.com/akabekobeko/redmine-theme-minimalflat2/releases/download/v1.5.0/minimalflat2-1.5.0.zip
unzip minimalflat2-1.5.0.zip
```


### Install plugin
```
ln -s /usr/share/redmine/bin /usr/share/rubygems-integration/all/specifications/

mkdir /usr/share/redmine/plugins
cd /usr/share/redmine/plugins

git clone https://github.com/applewu/redmine_omniauth_gitlab
cd /usr/share/redmine
bundle install --without development test
bundle exec rake redmine:plugins NAME=redmine_omniauth_gitlab RAILS_ENV=production
ll /usr/share/redmine/public/plugin_assets/
service apache2 restart
```

