---
title: Install Openshift
url: https://devopstales.github.io/kubernetes/ansible-openshift-install/
date: 2019-03-12
keywords: ansible, openshift 3.11, openshift container platform, rad hat openshift, rad hat, configuration management
---


Ansible-openshift is a pre made ansible playbook for Openshift installation. In this Post I will show you how to use to install a new Openshift cluster.

<!--more-->

{{< content "/filedir/openshift.html" >}}

### Environment
```
192.168.1.40    deployer
192.168.1.41    openshift01 # master node
192.168.1.42    openshift02 # infra node
192.168.1.43    openshift03 # worker node

# hardware requirement
4 CPU
16G RAM
```

### DNS config
```
master.openshift     300 IN  A 192.168.1.41
openshift            300 IN  A 192.168.1.42
*.openshift            300 IN  A 192.168.1.42
```

### Prerequirement
```
# deployer
yum install epel-release centos-release-openshift-origin311
yum --disablerepo=* --enablerepo=centos-ansible26 install ansible
yum install openshift-ansible nano

echo "exclude=ansible" >> /etc/yum.conf

nano ~/.ssh/config
Host openshift01
    Hostname openshift01.devopstales.intra
    User origin

Host openshift02
    Hostname openshift02.devopstales.intra
    User origin

Host openshift03
    Hostname openshift03.devopstales.intra
    User origin
```

```
# on all openshift hosts
hostnamectl set-hostname openshift01
yum -y update
yum -y install centos-release-openshift-origin311 epel-release docker git pyOpenSSL

useradd origin
passwd origin
echo -e 'Defaults:origin !requiretty\norigin ALL = (root) NOPASSWD:ALL' | tee /etc/sudoers.d/origin
chmod 440 /etc/sudoers.d/origin
reboot

# Disable swap permanently
nano /etc/fstab
#/dev/mapper/centos_openshift01-swap swap                    swap    defaults        0 0

sudo swapoff -a

sudo lvremove -Ay /dev/centos/swap
sudo lvextend -l +100%FREE centos/root
sudo xfs_growfs /

sudo nano /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
# GRUB_CMDLINE_LINUX="rd.lvm.lv=centos/root rd.lvm.lv=centos/swap crashkernel=auto rhgb quiet"
GRUB_CMDLINE_LINUX="rd.lvm.lv=centos/root crashkernel=auto rhgb quiet"
GRUB_DISABLE_RECOVERY="true"

dracut --regenerate-all -f
grub2-mkconfig -o /boot/grub2/grub.cfg
```

### Configurate Installer
```
# deployer

nano /etc/ansible/hosts
[OSEv3:children]
masters
nodes
etcd

[OSEv3:vars]
# admin user created in previous section
ansible_ssh_user=origin
ansible_become=true
openshift_deployment_type=origin
os_firewall_use_firewalld=True
openshift_clock_enabled=true

# use HTPasswd for authentication
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]

# define default sub-domain for Master node
openshift_master_default_subdomain=openshift.devopstales.intra
osm_default_subdomain=openshift.devopstales.intra

# allow unencrypted connection within cluster
openshift_docker_insecure_registries=172.30.0.0/16

openshift_master_cluster_hostname=master.openshift.devopstales.intra
openshift_master_cluster_public_hostname=master.openshift.devopstales.intra
openshift_public_hostname=master.openshift.devopstales.intra

openshift_master_api_port=443
openshift_master_console_port=443

[masters]
openshift01 containerized=true openshift_public_hostname=master.openshift.devopstales.intra

[etcd]
openshift01 containerized=true

[nodes]
# defined values for [openshift_node_group_name] in the file below
# [/usr/share/ansible/openshift-ansible/roles/openshift_facts/defaults/main.yml]
openshift01 openshift_node_group_name='node-config-master'
openshift02 openshift_node_group_name='node-config-infra'
openshift03 openshift_node_group_name='node-config-compute'
```

### Run the Installer
```
# deployer
cd /usr/share/ansible/openshift-ansible/
sudo ansible-playbook playbooks/prerequisites.yml
sudo ansible-playbook playbooks/deploy_cluster.yml

# If installastion failed or went wrong, the following uninstallation script can be run, and running installation can be tried again:
sudo ansible-playbook playbooks/adhoc/uninstall.yml
```

### User management
```
# on openshift master

cd /etc/origin/master/
# add user
htpasswd [/path/to/users.htpasswd] [user_name]
htpasswd htpasswd devopstales

# delete user
htpasswd -D [htpasswd/file/path/]  [user-name] [password]
htpasswd -D htpasswd devopstales Password1

# it will remove only the username from the htpasswd file by default it won’t remove user identity
oc delete  identity htpasswd_auth:user
```

