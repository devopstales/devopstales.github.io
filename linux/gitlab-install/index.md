---
title: Gitlab Install
url: https://devopstales.github.io/linux/gitlab-install/
date: 2019-06-05
keywords: postgresql, Gitlab, CentOS
---


Install Gitab with custom postgresql and nginx proxy.

<!--more-->
### Install NTPD
```
yum install -y epel-release yum-utils
yum-config-manager --enable epel

sudo chkconfig ntpd on
sudo ntpdate 0.hu.pool.ntp.org
sudo service ntpd start
```


### Install postgresql
In a previous post I wrote about how to [Install PostgreSQL 10]({{< relref "/linux/install-postgresql.md#install-postgresql-10-on-centos-7" >}})

```
su - postgres
createdb -U postgres gitlab
createdb -U postgres gitlab_ci
createdb -U postgres mattermost

createuser gituser
createuser ciuser
createuser mmuser

psql -U postgres gitlab
ALTER USER "gituser" WITH PASSWORD 'Password1';
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO gituser;
ALTER ROLE gituser CREATEROLE SUPERUSER;
\q

psql -U postgres gitlab_ci
ALTER USER "ciuser" WITH PASSWORD 'Password1';
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO ciuser;
\q

psql -U postgres mattermost
ALTER USER "mmuser" WITH PASSWORD 'Password1';
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO mmuser;
\q
```

### install nginx
```
yum install -y nginx
mkdir /var/log/gitlab/nginx/

echo 'upstream gitlab-workhorse {
        server unix:/var/opt/gitlab/gitlab-workhorse/socket;
}

server {
        listen *:80;
#  listen *:443 ssl;
        server_name gitlab.devopstales.intra;
        server_tokens off;
        root /opt/gitlab/embedded/service/gitlab-rails/public;

        client_max_body_size 256m;

        real_ip_header X-Forwarded-For;

        access_log /var/log/gitlab/nginx/gitlab_access.log;
        error_log  /var/log/gitlab/nginx/gitlab_error.log;

 # ssl_certificate   /etc/nginx/ssl.d/gitlab.pem;
 # ssl_certificate_key   /etc/nginx/ssl.d/gitlab.key;


        location / {
                proxy_read_timeout	300;
                proxy_connect_timeout   300;
                proxy_redirect          off;

                proxy_buffering off;

                proxy_set_header    Host                $http_host;
                proxy_set_header    X-Real-IP           $remote_addr;
                proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
                proxy_set_header    X-Forwarded-Proto   $scheme;

                proxy_pass http://gitlab-workhorse;

    proxy_request_buffering off;
    proxy_http_version 1.1;
        }

        location ~ ^/(assets)/ {
                root /opt/gitlab/embedded/service/gitlab-rails/public;
                gzip_static on; # to serve pre-gzipped version
                        expires max;
               	add_header Cache-Control public;
        }

        error_page 502 /502.html;
}' > /etc/nginx/conf.d/gitlab.conf

nano /etc/nginx/nginx.conf
log_format gitlab_access '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"';
log_format gitlab_ci_access '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"';
log_format gitlab_mattermost_access '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"';

nginx -t
nginx -s reload

sudo usermod -aG gitlab-www nginx
```

### install gitlab
```
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
sudo yum install gitlab-ce -y

cd /opt/gitlab/embedded/bin
mv psql psql_moved
mv pg_dump pg_dump_moved

which pg_dump psql

ln -s /usr/bin/pg_dump /usr/bin/psql /opt/gitlab/embedded/bin/
```

```
nano /etc/gitlab/gitlab.rb
external_url 'http://gitlab.devopstales.intra'
gitlab_rails['time_zone'] = 'Europe/Budapest'

gitlab_rails['db_adapter'] = "postgresql"
gitlab_rails['db_encoding'] = "unicode"
gitlab_rails['db_database'] = "gitlab"
gitlab_rails['db_pool'] = 10
gitlab_rails['db_username'] = "gituser"
gitlab_rails['db_password'] = "Password1"
gitlab_rails['db_host'] = '127.0.0.1'
gitlab_rails['db_port'] = 5432

gitlab_rails['redis_socket'] = "/var/opt/gitlab/redis/redis.socket"

unicorn['socket'] = '/var/opt/gitlab/gitlab-rails/sockets/gitlab.socket'
unicorn['log_directory'] = "/var/log/gitlab/unicorn"
unicorn['port'] = 8081

user['username'] = "git"
user['group'] = "git"

postgresql['enable'] = false

nginx['enable'] = false
web_server['external_users'] = ['nginx']
```

```
gitlab-ctl reconfigure
```

