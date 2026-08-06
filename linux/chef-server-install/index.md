---
title: Install  Chef server
url: https://devopstales.github.io/linux/chef-server-install/
date: 2020-04-30
keywords: chef-server, configuration management
---

Chef is a powerful configuration management utility writy in ruby. This post will help you to setup a chef 13 on CentOS 7
<!--more-->

* Chef Server: This is the central hub server that stores the cookbooks and recipes uploaded from workstations.
* Chef Workstations: This where recipes, cookbooks, and other chef configuration details are created or edited.
* Chef Client: This the target node where the configurations are deployed by the chef-client.

## Chef Server Install:
```
cd /opt
wget https://packages.chef.io/files/stable/chef-server/13.2.0/el/7/chef-server-core-13.2.0-1.el7.x86_64.rpm
yum install chef-server-core-13.2.0-1.el7.x86_64.rpm -y

chef-server-ctl reconfigure
chef-server-ctl status
```

Create admin user for chef server:
```
# chef-server-ctl user-create USER_NAME FIRST_NAME LAST_NAME EMAIL 'PASSWORD' -f PATH_FILE_NAME
chef-server-ctl user-create admin admin admin admin@devopstales.intra Password1 -f /etc/chef/admin.pem
```

Now create an organization to hold the chef configurations.
```
# chef-server-ctl org-create short_name 'full_organization_name' --association_user user_name --filename ORGANIZATION-validator.pem

chef-server-ctl org-create devopstales "DevOpsTales, Inc" --association_user admin -f /etc/chef/devopstales-validator.pem
```

## Install Chef workstation:
Fot this demo I will install the workstation on the same server as the Chef server, but in a pruduction enviroment it is your laptop or pc.

```
wget https://packages.chef.io/files/stable/chefdk/4.7.73/el/7/chefdk-4.7.73-1.el7.x86_64.rpm
yum install -y chefdk-4.7.73-1.el7.x86_64.rpm
chef verify
```

```
which ruby
echo 'eval "$(chef shell-init bash)"' >> ~/.bash_profile
. ~/.bash_profile
which ruby
```

```
cd ~
chef generate repo chef-repo
mkdir -p ~/chef-repo/.chef
cp /etc/chef/admin.pem ~/chef-repo/.chef/
cp /etc/chef/devopstales-validator.pem ~/chef-repo/.chef/
```

```
nano ~/chef-repo/.chef/knife.rb
current_dir = File.dirname(__FILE__)
log_level                :info
log_location             STDOUT
node_name                "admin"
client_key               "#{current_dir}/admin.pem"
validation_client_name   "devopstzales-validator"
validation_key           "#{current_dir}/itzgeek-validator.pem"
chef_server_url          "https://cchef.mydomain.intra/organizations/devopstales"
syntax_check_cache_path  "#{ENV['HOME']}/.chef/syntaxcache"
cookbook_path            ["#{current_dir}/../cookbooks"]
```

test kinife client:
```
cd ~/chef-repo/
knife ssl fetch
knife client list
```

## Install chef client:
Before we can bootstrap a chef client on a server we need valid DNS resolution for both.
```
knife bootstrap -N test.mydomain.intra test.mydomain.intra -y root -P vagrant
```

---
* https://downloads.chef.io/chef-server/stable/
* https://downloads.chef.io/chefdk/stable/

