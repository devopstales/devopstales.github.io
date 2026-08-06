---
title: Openshift LDAP authentication
url: https://devopstales.github.io/kubernetes/openshift-ldap/
date: 2019-04-12
keywords: ansible, openshift 3.11, openshift okd, openshift tutorial, openshift container platform, rad hat openshift, open ldap
---


Configure Openshift Cluster to use LDAP as a user backend for login with Ansible-openshift

<!--more-->

{{< content "/filedir/openshift.html" >}}

In the last post I used the basic htpasswd authentication method for the installatipn.<br>
But I can use Ansible-openshift to configure an LDAP backed at the install for the authentication.

### Environment
```
192.168.1.40    deployer
192.168.1.41    openshift01 # master node
192.168.1.42    openshift02 # infra node
192.168.1.43    openshift03 # worker node
```

With Ansible-openshift you can not change the authetication method after Install !! If you installed the cluster with htpasswd, then change to LDAP the playbook trys to add a second authentication methot for the config. It is forbidden to add a second type of identity provider in the version 3.11 of Ansible-openshift so choose wisely.

### Configurate Installer
```
# deployer
nano /etc/ansible/ansible.cfg
# use HTPasswd for authentication
#openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]

# LDAP
openshift_master_identity_providers=[{'name': 'email_jira_ldap', 'challenge': 'true', 'login': 'true', 'kind': 'LDAPPasswordIdentityProvider', 'attributes': {'id': ['mail'], 'email': ['mail'], 'name': ['displayName'], 'preferredUsername': ['mail']}, 'bindDN': 'CN=ldapbrowser,DC=mydomain,DC=myintra', 'bindPassword': '*******', 'insecure': 'true', 'url': 'ldap://ldap01.mydomain.myintra/dc=mydomain,dc=myintra?mail?sub?(objectClass=*)'}]
```

### Run the Installer
```
# deployer
cd /usr/share/ansible/openshift-ansible/
sudo ansible-playbook -i inventory/hosts.localhost playbooks/prerequisites.yml
sudo ansible-playbook -i inventory/hosts.localhost playbooks/deploy_cluster.yml
```

