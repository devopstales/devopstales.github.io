---
title: Wazuh SIEM Authentication
url: https://devopstales.github.io/linux/wazuh-authentication/
date: 2023-09-05
keywords: Wazuh, SIEM, logging, CentOS
---


In this post I will show you how to configure LDAP Authentication in a Wazuh Open Source SIEM solution.
<!--more-->

Wazuh SIEM is base on Opensearch the fork of Elasticsearch. So we need to edit the security configuration for the Wazuh Indexer that is basicly an Opensearch.

```yaml
nano /etc/wazuh-indexer/opensearch-security/config.yml
---
config:
  dynamic:
    authc:
      basic_internal_auth_domain:
       	description: "Authenticate via HTTP Basic against internal users database"
       	http_enabled: true
        transport_enabled: true
       	order: 4
       	http_authenticator:
          type: basic
          challenge: true
        authentication_backend:
          type: intern
      ldap:
	description: "Authenticate via LDAP or Active Directory"
        http_enabled: true
        transport_enabled: true
        order: 1
        http_authenticator:
          type: basic
          challenge: false
        authentication_backend:
          type: ldap
          config:
            enable_ssl: false
            enable_start_tls: false
            enable_ssl_client_auth: false
            verify_hostnames: true
            hosts:
            - 192.168.0.40:389
            bind_dn: CN=ldapsync,DC=mydomain,DC=intra
            password: Aa123456
            userbase: 'DC=mydomain,DC=intra'
            usersearch: '(sAMAccountName={0})'
            username_attribute: '(sAMAccountName={0})'
    authz:
      roles_from_myldap:
        description: "Authorize via LDAP or Active Directory"
        http_enabled: true
        transport_enabled: true
        authorization_backend:
          type: ldap
          config:
            enable_ssl: false
            enable_start_tls: false
            enable_ssl_client_auth: false
            verify_hostnames: true
            hosts:
            - 192.168.0.40:389
            bind_dn: CN=ldapsync,DC=mydomain,DC=intra
            password: Aa123456
            rolebase: 'DC=mydomain,DC=intra'
            rolesearch: '(member={0})'
            userroleattribute: null
            userrolename: none
            rolename: cn
            resolve_nested_roles: true
            userbase: 'DC=mydomain,DC=intra'
            usersearch: '(sAMAccountName={0})'
            skip_users:
            - kibanaserver
```

Create role mapping for your group. In this example I will use `devops-team` to mapp to the same role as admin.

```yaml
nano /etc/wazuh-indexer/opensearch-security/roles_mapping.yml
---
all_access:
  reserved: true
  hidden: false
  backend_roles:
  - "admin"
  - "devops-team"
  hosts: []
  users: []
  and_backend_roles: []
  description: "Maps admin to all_access"
```

Apply config and role mapping:

```bash
export JAVA_HOME=/usr/share/wazuh-indexer/jdk/
bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
	-f /etc/wazuh-indexer/opensearch-security/config.yml -icl \
        -key /etc/wazuh-indexer/certs/admin-key.pem \
        -cert /etc/wazuh-indexer/certs/admin.pem \
        -cacert /etc/wazuh-indexer/certs/root-ca.pem \
        -h 192.168.108.102 -nhnv

bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
	-f /etc/wazuh-indexer/opensearch-security/roles_mapping.yml -icl \
	-key /etc/wazuh-indexer/certs/admin-key.pem \
	-cert /etc/wazuh-indexer/certs/admin.pem \
	-cacert /etc/wazuh-indexer/certs/root-ca.pem \
	-h 192.168.108.102 -nhnv

systemctl restart wazuh-dashboard
```

