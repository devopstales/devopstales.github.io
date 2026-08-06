---
title: Install Katello
url: https://devopstales.github.io/linux/katello-install/
date: 2019-04-29
---


Katello brings the full power of content management alongside the provisioning and configuration capabilities of Foreman. Katello is the upstream community project from which the Red Hat Satellite product is derived after Red Hat Satellite Server 6.
<!--more-->

### Base komponents


* Foreman: provisioning on new clients.
* Pulp: patch and content (package repository) management.
* Candlepin: subscription and entitlement management.
* Puppet: configuration management (actual running of modules assigned in Foreman).
* Katello: unified workflow and WebUI for content (Pulp) and subscriptions (Candlepin).


### Hardware Requirements

* Two Logical CPUs
* 8 GB of memory (12 GB highly recommended)
* The filesystem holding /var/lib/pulp needs to be large


### Required Repositories
```
# hostnevet beállítani !!!

yum -y localinstall https://fedorapeople.org/groups/katello/releases/yum/3.11/katello/el7/x86_64/katello-repos-latest.rpm
yum -y localinstall https://yum.theforeman.org/releases/1.21/el7/x86_64/foreman-release.rpm
yum -y localinstall https://yum.puppetlabs.com/puppetlabs-release-pc1-el-7.noarch.rpm
yum -y localinstall https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
```

### Installation
```bash
yum -y install foreman-release-scl python-django
yum -y update
yum -y install katello


foreman-installer \
--scenario "katello" \
--foreman-initial-organization "mydomain" \
--foreman-initial-location "office" \
--enable-foreman-plugin-ansible \
--enable-foreman-proxy-plugin-ansible \
--enable-foreman-plugin-remote-execution \
--enable-foreman-proxy-plugin-remote-execution-ssh

# reset/gen Password
foreman-rake permissions:reset
```

### Configure hammer-cli
```
nano ~/.hammer/cli.modules.d/foreman.yml
:foreman:
 :host: 'https://katello.devopstales.intra/'
 :username: 'admin'
 :password: '**********'

hammer defaults add --param-name organization --param-value "mydomain"
hammer defaults add --param-name location --param-value "office"
hammer defaults list
```

### Configure gpg keys
```
hammer product create \
--name "el7_repos" \
--description "Various repositories to use with CentOS 7"

mkdir /etc/pki/rpm-gpg/import/
cd /etc/pki/rpm-gpg/import/
wget https://repo.mysql.com/RPM-GPG-KEY-mysql
wget http://mirror.centos.org/centos/7/os/x86_64/RPM-GPG-KEY-CentOS-7
wget https://archive.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-7Server
wget https://rpms.remirepo.net/RPM-GPG-KEY-remi
wget https://packages.cisofy.com/keys/cisofy-software-rpms-public.key

hammer gpg create \
--key "RPM-GPG-KEY-CentOS-7" \
--name "RPM-GPG-KEY-CentOS-7"

hammer gpg create \
--key "RPM-GPG-KEY-mysql" \
--name "RPM-GPG-KEY-mysql"

hammer gpg create \
--key "RPM-GPG-KEY-EPEL-7Server" \
--name "RPM-GPG-KEY-EPEL-7Server"

hammer gpg create \
--key "RPM-GPG-KEY-remi" \
--name "RPM-GPG-KEY-remi"

hammer gpg create \
--key "cisofy-software-rpms-public.key" \
--name "RPM-GPG-KEY-cisofy"
```

### Create yum repositories
```
hammer gpg list

hammer repository create \
--product "el7_repos" \
--name "base_x86_64" \
--label "base_x86_64" \
--content-type "yum" \
--download-policy "on_demand" \
--gpg-key "RPM-GPG-KEY-CentOS-7" \
--url "http://mirror.centos.org/centos/7/os/x86_64/" \
--mirror-on-sync "no"

hammer repository create \
--product "el7_repos" \
--name "extras_x86_64" \
--label "extras_x86_64" \
--content-type "yum" \
--download-policy "on_demand" \
--gpg-key "RPM-GPG-KEY-CentOS-7" \
--url "http://mirror.centos.org/centos/7/extras/x86_64/" \
--mirror-on-sync "no"

hammer repository create \
--product "el7_repos" \
--name "updates_x86_64" \
--label "updates_x86_64" \
--content-type "yum" \
--download-policy "on_demand" \
--gpg-key "RPM-GPG-KEY-CentOS-7" \
--url "http://mirror.centos.org/centos/7/updates/x86_64/" \
--mirror-on-sync "no"

hammer repository create \
--product "el7_repos" \
--name "epel_x86_64" \
--label "epel_x86_64" \
--content-type "yum" \
--download-policy "on_demand" \
--gpg-key "RPM-GPG-KEY-EPEL-7Server" \
--url "https://dl.fedoraproject.org/pub/epel/7Server/x86_64/"

hammer repository create \
--product "el7_repos" \
--name "lynis" \
--label "lynis" \
--content-type "yum" \
--download-policy "on_demand" \
--gpg-key "RPM-GPG-KEY-cisofy" \
--url "https://packages.cisofy.com/community/lynis/rpm/"

hammer repository create \
--product "el7_repos" \
--name "mysql_57_x86_64" \
--label "mysql_57_x86_64" \
--content-type "yum" \
--download-policy "on_demand" \
--gpg-key "RPM-GPG-KEY-mysql" \
--url "https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/"

hammer repository create \
--product "el7_repos" \
--name "katello_agent_x86_64" \
--label "katello_agent_x86_64" \
--content-type "yum" \
--download-policy "on_demand" \
--url "https://fedorapeople.org/groups/katello/releases/yum/latest/client/el7/x86_64/"

hammer repository create \
--product "el7_repos" \
--name "remi_php_56_x86_64" \
--label "remi_php_56_x86_64" \
--content-type "yum" \
--download-policy "on_demand" \
--gpg-key "RPM-GPG-KEY-remi" \
--url "https://mirrors.ukfast.co.uk/sites/remi/enterprise/7/php56/x86_64/"

hammer repository create \
--product "el7_repos" \
--name "remi_php_72_x86_64" \
--label "remi_php_72_x86_64" \
--content-type "yum" \
--download-policy "on_demand" \
--gpg-key "RPM-GPG-KEY-remi" \
--url "https://mirrors.ukfast.co.uk/sites/remi/enterprise/7/php72/x86_64/"

hammer repository create \
--product "el7_repos" \
--name "remi_safe_x86_64" \
--label "remi_safe_x86_64" \
--content-type "yum" \
--download-policy "on_demand" \
--gpg-key "RPM-GPG-KEY-remi" \
--url "https://mirrors.ukfast.co.uk/sites/remi/enterprise/7/safe/x86_64/"
```

### Sync repos
```
hammer repository list

for i in $(seq 1 12); do \
hammer repository synchronize \
--product "el7_repos" \
--id "$i"; \
done

# Create a Content View
hammer content-view create \
--name "el7_content" \
--description "Content view for CentOS 7"

hammer product list

# Add Repositories to Content View
for i in $(seq 1 12); do \
hammer content-view add-repository \
--name "el7_content" \
--product "el7_repos" \
--repository-id "$i"; \
done

# Create a Lifecycle Environment
hammer lifecycle-environment create \
--name "stable" \
--label "stable" \
--prior "Library"

hammer lifecycle-environment list

# Publish a Content View
hammer content-view publish \
--name "el7_content" \
--description "Publishing repositories"

hammer content-view version list

# Promote Version to Lifecycle Environment
hammer content-view version promote \
--content-view "el7_content" \
--version "1.0" \
--to-lifecycle-environment "stable"

hammer content-view version list

# Create an Activation Key
hammer activation-key create \
--name "el7-key" \
--description "Key to use with CentOS7" \
--lifecycle-environment "stable" \
--content-view "el7_content" \
--unlimited-hosts

hammer activation-key list

# Add Subscription to Activation Key
hammer subscription list

hammer activation-key add-subscription \
--name "el7-key" \
--quantity "1" \
--subscription-id "1"

# Backup Katello Configuration
foreman-maintain backup snapshot -y /mnt/backup/
```

