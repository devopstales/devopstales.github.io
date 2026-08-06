---
title: Configure Rundeck LADAP
url: https://devopstales.github.io/linux/rundeck-ldap/
date: 2019-12-09
---


In this post I will configure Rundeck to use LDAP as a User backend.

<!--more-->

### Rundeck LDAP config file
```
nano /etc/rundeck/jaas-ldap.conf

# openldap
ldap {
      com.dtolabs.rundeck.jetty.jaas.JettyCachingLdapLoginModule required
      contextFactory="com.sun.jndi.ldap.LdapCtxFactory"
      debug="true"
      providerUrl="ldap://openldap:389"
      bindDn="cn=admin,dc=mydomain,dc=intra"
      bindPassword="Password1"
      authenticationMethod="simple"
      forceBindingLogin="true"
      userBaseDn="dc=mydomain,dc=intra"
      userRdnAttribute="cn"
      userIdAttribute="cn"
      userPasswordAttribute="userPassword"
      userObjectClass="inetOrgPerson"
      roleBaseDn="dc=mydomain,dc=intra"
      roleNameAttribute="cn"
      roleMemberAttribute="uniqueMember"
      roleObjectClass="groupOfUniqueNames"
      supplementalRoles="admin, user";
      };

# windows AD
ldap {
      com.dtolabs.rundeck.jetty.jaas.JettyCachingLdapLoginModule required
      contextFactory="com.sun.jndi.ldap.LdapCtxFactory"
      debug="true"
      providerUrl="ldap://devopstales.intra:389"
      bindDn="cn=admin,dc=mydomain,dc=intra"
      bindPassword="Password1"
      authenticationMethod="simple"
      forceBindingLogin="true"
      userBaseDn="dc=mydomain,dc=intra"
      userRdnAttribute="sAMAccountName"
      userIdAttribute="sAMAccountName"
      userPasswordAttribute="unicodePwd"
      userObjectClass="user"
      roleBaseDn="dc=mydomain,dc=intra"
      roleNameAttribute="cn"
      roleMemberAttribute="member"
      roleObjectClass="group"
      supplementalRoles="admin, user";
      };
```

### Rundeck multibackend config file
```
nano /etc/rundeck/jaas-multiauth.conf

multiauth {
      com.dtolabs.rundeck.jetty.jaas.JettyCombinedLdapLoginModule required
      contextFactory="com.sun.jndi.ldap.LdapCtxFactory"
      debug="true"
      providerUrl="ldap://ad1:389"
      bindDn="cn=admin,dc=mydomain,dc=intra"
      bindPassword="Password1"
      authenticationMethod="simple"
      forceBindingLogin="true"
      userBaseDn="ou=Users,dc=mydomain,dc=intra"
      userRdnAttribute="cn"
      userIdAttribute="cn"
      userPasswordAttribute="userPassword"
      userObjectClass="inetOrgPerson"
      roleBaseDn="ou=Groups,dc=mydomain,dc=intra"
      roleNameAttribute="cn"
      roleMemberAttribute="uniqueMember"
      roleObjectClass="groupOfUniqueNames"

      ignoreRoles="true"
      storePass="true"
      clearPass="true"
      useFirstPass="false"
      tryFirstPass="false"
      supplementalRoles="admin, user";

      org.rundeck.jaas.jetty.JettyRolePropertyFileLoginModule required
      debug="true"
      useFirstPass="true"
      file="/etc/rundeck/realm.properties";
      };
```

### Configure Rundeck to use multibackend config file
```
nano /etc/rundeck/profile

JAAS_CONF="${JAAS_CONF:-$RDECK_CONFIG/jaas-ldap.conf}"
LOGIN_MODULE="ldap"

# OR based on your distro
nano /etc/default/rundeckd

export JAAS_CONF="/etc/rundeck/jaas-ldap.conf"
export LOGIN_MODULE="ldap"
```

### Configure Rundeck to use LDAP config file
```
nano /etc/rundeck/profile

JAAS_CONF="${JAAS_CONF:-$RDECK_CONFIG/jaas-multiauth.conf}"
LOGIN_MODULE="multiauth"

# OR based on your distro
nano /etc/default/rundeckd

export JAAS_CONF="/etc/rundeck/jaas-multiauth.conf"
export LOGIN_MODULE="multiauth"
```

