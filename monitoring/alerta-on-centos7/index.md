---
title: Install alerta on Centos7
url: https://devopstales.github.io/monitoring/alerta-on-centos7/
date: 2019-09-27
keywords: alerta, postgresql, postgres, pgsql, centos 7
---


AIn this post I will show you how to install alerta monitoring dashboard on Centos 7.

<!--more-->

```
yum install epel-release nano -y
yum upgrade -y
```

### Install and configure postgresql

In a previous post I wrote about how to [Install PostgreSQL 10]({{< relref "/linux/install-postgresql.md#install-postgresql-10-on-centos-7" >}})

```
sudo su - postgres
createuser alerta
createdb -O alerta alerta
psql
ALTER USER "alerta" WITH PASSWORD 'alerta';
\q
```

### install python3 packages
```
yum install https://centos7.iuscommunity.org/ius-release.rpm -y
yum install python36u python36u-pip python36u-setuptools python36u-devel gcc git tmux
yum install uwsgi-plugin-psgi nginx -y
```

# alerta server
```
python3.6 -m venv alerta
alerta/bin/pip install --upgrade pip wheel alerta-server alerta uwsgi
```

```
nano /etc/alertad.conf
DATABASE_URL = 'postgres://alerta:alerta@localhost:5432/alerta'
PLUGINS=['reject']
ALLOWED_ENVIRONMENTS=['Production', 'Development', 'Code']
```

We use uwsgi to create a unix socket wgere the alerta webgui can connect.
```
sudo mkdir -p /var/log/uwsgi
sudo chown -R nginx:nginx /var/log/uwsgi
mkdir /var/run/alerta
chown -R nginx.nginx /var/run/alerta/

nano /var/www/wsgi.py
from alerta import app
```

```
nano /etc/uwsgi.ini
[uwsgi]
chdir = /var/www
mount = /api=wsgi.py
callable = app
manage-script-name = true
env = BASE_URL=/api

master = true
processes = 5
#logger = syslog:alertad
logto = /var/log/uwsgi/%n.log

socket = /var/run/alerta/uwsgi.sock
chmod-socket = 664
uid = nginx
gid = nginx
vacuum = true

die-on-term = true
```

```
nano /etc/systemd/system/alerta-app.service
[Unit]
Description=uWSGI service
After=syslog.target

[Service]
ExecStart=/opt/alerta/bin/uwsgi --ini /etc/uwsgi.ini
RuntimeDirectory=uwsgi
Type=notify
StandardError=syslog
NotifyAccess=all

[Install]
WantedBy=multi-user.target
```

```
systemctl enable alerta-app
systemctl start alerta-app
systemctl status alerta-app
```

# alerta webgui
```
wget https://github.com/alerta/alerta-webui/releases/latest/download/alerta-webui.tar.gz
tar zxvf alerta-webui.tar.gz
mv dist/ /usr/share/nginx/alerta
echo '{"endpoint": "/api"}' > /usr/share/nginx/dist/config.json
```

```
nano /etc/nginx/nginx.conf
    #server {
    #    listen       80 default_server;
    #    listen       [::]:80 default_server;
    #    server_name  _;
    #    root         /usr/share/nginx/html;

    #    # Load configuration files for the default server block.
    #    include /etc/nginx/default.d/*.conf;

     #   location / {
     #   }

     #   error_page 404 /404.html;
     #       location = /40x.html {
     #   }

     #   error_page 500 502 503 504 /50x.html;
     #       location = /50x.html {
     #   }
    #}
```

```
nano /etc/nginx/conf.d/alerta.conf
server {
        listen 80 default_server;

        location /api { try_files $uri @api; }
        location @api {
            include uwsgi_params;
            uwsgi_pass unix:/var/run/alerta/uwsgi.sock;
            proxy_set_header Host $host:$server_port;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location / {
                root /usr/share/nginx/alerta;
        }
}
```

```
nginx -t
systemctl restart nginx
systemctl enable nginx
```

### test
Send testalert with alerta client.
```
nano $HOME/.alerta.conf
[DEFAULT]
endpoint = http://localhost/api

/opt/alerta/bin/alerta query
/opt/alerta/bin/alerta send --resource net01 --event down --severity critical --environment Code --service Network --text 'net01 is down.'
/opt/alerta/bin/alerta query
```

