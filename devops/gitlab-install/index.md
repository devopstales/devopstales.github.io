---
title: GitLab Installation Guide
url: https://devopstales.github.io/devops/gitlab-install/
date: 2026-03-09
keywords: GitLab, CI/CD, DevOps
---


Install GitLab CE with custom PostgreSQL and Nginx proxy.

<!--more-->

### Prerequisites

* AlmaLinux/Rocky Linux 9 or Ubuntu 22.04+
* Minimum 4GB RAM (8GB recommended)
* 2+ CPU cores
* PostgreSQL 15+ (included or external)

### System Time Synchronization

Modern Linux distributions use `chrony` instead of `ntpd`:

```bash
# AlmaLinux/Rocky Linux 9
dnf install -y chrony
systemctl enable --now chronyd
timedatectl set-timezone Europe/Budapest
chronyc sources -v
```


### Install PostgreSQL

GitLab 18.8 requires PostgreSQL 15+. You can use the bundled PostgreSQL or install externally.

#### External PostgreSQL Installation (Recommended for Production)

```bash
# Install PostgreSQL 15
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf install -y postgresql15 postgresql15-server postgresql15-contrib

# Initialize and start PostgreSQL
postgresql-15-setup --initdb
systemctl enable --now postgresql-15

# Create databases and users
su - postgres
psql -c "CREATE DATABASE gitlabhq_production OWNER gitlab ENCODING 'UTF8';"
psql -c "CREATE DATABASE gitlab_ci_production OWNER gitlab ENCODING 'UTF8';"
psql -c "CREATE USER gitlab WITH PASSWORD 'SecurePassword123!';"
psql -c "GRANT ALL PRIVILEGES ON DATABASE gitlabhq_production TO gitlab;"
psql -c "GRANT ALL PRIVILEGES ON DATABASE gitlab_ci_production TO gitlab;"
psql -c "\\q"
exit
```

### Install Nginx

```bash
dnf install -y nginx
mkdir -p /var/log/gitlab/nginx/
mkdir -p /etc/nginx/conf.d/

cat > /etc/nginx/conf.d/gitlab.conf << 'EOF'
upstream gitlab-workhorse {
    server unix:/var/opt/gitlab/gitlab-workhorse/socket;
}

server {
    listen *:80;
    server_name gitlab.devopstales.intra;
    server_tokens off;
    root /opt/gitlab/embedded/service/gitlab-rails/public;

    client_max_body_size 256m;

    real_ip_header X-Forwarded-For;

    access_log /var/log/gitlab/nginx/gitlab_access.log;
    error_log  /var/log/gitlab/nginx/gitlab_error.log;

    location / {
        proxy_read_timeout    300;
        proxy_connect_timeout 300;
        proxy_redirect        off;
        proxy_buffering       off;

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
        gzip_static on;
        expires max;
        add_header Cache-Control public;
    }

    error_page 502 /502.html;
}
EOF

# Test and reload Nginx
nginx -t
systemctl enable --now nginx
systemctl reload nginx

# Add nginx user to gitlab-www group
usermod -aG gitlab-www nginx
```

### Install GitLab

```bash
# Add GitLab repository
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash

# Install GitLab CE (specify version if needed)
dnf install -y gitlab-ce

# Stop bundled PostgreSQL if using external
systemctl stop postgresql-15

# Create symlinks for PostgreSQL client tools (if needed)
cd /opt/gitlab/embedded/bin
mv psql psql_moved 2>/dev/null || true
mv pg_dump pg_dump_moved 2>/dev/null || true
ln -s /usr/bin/pg_dump /usr/bin/psql /opt/gitlab/embedded/bin/ 2>/dev/null || true
cd -
```

Configure GitLab:

```ruby
nano /etc/gitlab/gitlab.rb
---
external_url 'http://gitlab.devopstales.intra'
gitlab_rails['time_zone'] = 'Europe/Budapest'

# External PostgreSQL configuration
gitlab_rails['db_adapter'] = "postgresql"
gitlab_rails['db_encoding'] = "unicode"
gitlab_rails['db_database'] = "gitlabhq_production"
gitlab_rails['db_pool'] = 10
gitlab_rails['db_username'] = "gitlab"
gitlab_rails['db_password'] = "SecurePassword123!"
gitlab_rails['db_host'] = '127.0.0.1'
gitlab_rails['db_port'] = 5432

# Redis (use bundled)
gitlab_rails['redis_socket'] = "/var/opt/gitlab/redis/redis.socket"

# Unicorn/Puma configuration
puma['worker_processes'] = 2
puma['min_threads'] = 1
puma['max_threads'] = 16

# User settings
user['username'] = "git"
user['group'] = "git"

# Disable bundled PostgreSQL
postgresql['enable'] = false

# Nginx settings
nginx['enable'] = false
web_server['external_users'] = ['nginx']

# Security settings
gitlab_rails['rate_limit_requests_per_period'] = 10
gitlab_rails['rate_limit_period'] = 1
```

Apply configuration:

```bash
gitlab-ctl reconfigure
gitlab-ctl restart

# Check status
gitlab-ctl status
```

### Post-Installation

1. **Access GitLab**: Open http://gitlab.devopstales.intra in your browser
2. **Set root password**: The initial root password is stored at `/etc/gitlab/initial_root_password`
3. **Configure backups**: Add backup schedule to crontab
4. **Set up monitoring**: Configure Prometheus/Grafana for GitLab metrics

### Backup Configuration

```bash
# Create backup cron job
cat >> /etc/crontab << EOF
0 2 * * * root /opt/gitlab/bin/gitlab-backup create CRON=1
EOF

# Configure backup retention
echo "gitlab_rails['backup_keep_time'] = 604800" >> /etc/gitlab/gitlab.rb
gitlab-ctl reconfigure
```

