---
title: Install mattermost for Gitlab
url: https://devopstales.github.io/linux/install-mattermost-for-gitlab/
date: 2019-06-09
keywords: gitlab mattermost
---


 Mattermost is an open source on premise alternative of Slack.

<!--more-->

### Install mattermost from package
```
sudo yum -y install https://harbottle.gitlab.io/harbottle-main/7/x86_64/harbottle-main-release.rpm

yum install mattermost-server -y
```

### Configurate mattermost
```
nano /etc/mattermost/config.json
# postgresql
"SiteURL": "http://mattermost.devopstales.intra"
"DriverName": "postgres",
"DataSource": "postgres://mmuser:Password1@localhost:5432/mattermost?sslmode=disable&connect_timeout=10"

# gitlab austh
"AllowedUntrustedInternalConnections": "gitlab.devopstales.intra"
...
    "GitLabSettings": {
        "Enable": true,
        "Secret": "<secret>",
        "Id": "<id>",
        "Scope": "",
        "AuthEndpoint": "http://gitlab.devopstales.intra/oauth/authorize",
        "TokenEndpoint": "http://gitlab.devopstales.intra/oauth/token",
        "UserApiEndpoint": "http://gitlab.devopstales.intra/api/v4/user"
    },
```

### Edit systemd serice
```
nano /usr/lib/systemd/system/mattermost.service
[Unit]
Description=Mattermost
After=syslog.target network.target
After=postgresql.service
Requires=postgresql-9.6.service

[Service]
Type=notify
NotifyAccess=main
WorkingDirectory=/usr/share/mattermost
User=mattermost
Group=mattermost
ExecStart=/usr/share/mattermost/bin/mattermost
TimeoutStartSec=3600
LimitNOFILE=49152

[Install]
WantedBy=multi-user.target

systemctl daemon-reload
systemctl start mattermost.service
```

### Configurate nginx proxy
```
nano /etc/nginx/conf.d/mattermost.conf
upstream backend {
   server localhost:8065;
   keepalive 32;
}

proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mattermost_cache:10m max_size=3g inactive=120m use_temp_path=off;

server {
   listen 80;
   server_name    mattermost.devopstales.intra;

   location ~ /api/v[0-9]+/(users/)?websocket$ {
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
       client_max_body_size 50M;
       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
       proxy_set_header X-Frame-Options SAMEORIGIN;
       proxy_buffers 256 16k;
       proxy_buffer_size 16k;
       client_body_timeout 60;
       send_timeout 300;
       lingering_timeout 5;
       proxy_connect_timeout 90;
       proxy_send_timeout 300;
       proxy_read_timeout 90s;
       proxy_pass http://backend;
   }

   location / {
       client_max_body_size 50M;
       proxy_set_header Connection "";
       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
       proxy_set_header X-Frame-Options SAMEORIGIN;
       proxy_buffers 256 16k;
       proxy_buffer_size 16k;
       proxy_read_timeout 600s;
       proxy_cache mattermost_cache;
       proxy_cache_revalidate on;
       proxy_cache_min_uses 2;
       proxy_cache_use_stale timeout;
       proxy_cache_lock on;
       proxy_http_version 1.1;
       proxy_pass http://backend;
   }
}
```

