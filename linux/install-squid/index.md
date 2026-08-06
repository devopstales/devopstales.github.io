---
title: Install Squid proxy
url: https://devopstales.github.io/linux/install-squid/
date: 2019-11-18
keywords: proxy, Squid
---


Squid is the most popular Proxy server for Linux systems. The squid proxy server is also useful for the web packet filtering.
<!--more-->


### Install Squid
Squid packages are available in default yum repositories.

```
yum install squid
```

### Configuring Squid


```
nano /etc/squid/squid.conf
    #
    # Recommended minimum configuration:
    ## Example rule allowing access from your local networks.
    # Adapt to list your (internal) IP networks from where browsing
    # should be allowed
    acl localnet src 10.0.0.0/8     # RFC1918 possible internal network
    acl localnet src 172.16.0.0/12  # RFC1918 possible internal network
    acl localnet src 192.168.0.0/16 # RFC1918 possible internal network
    acl localnet src fc00::/7       # RFC 4193 local private network range
    acl localnet src fe80::/10      # RFC 4291 link-local (directly plugged)

    machinesacl SSL_ports port 443
    acl Safe_ports port 80          # http
    acl Safe_ports port 21          # ftp
    acl Safe_ports port 443         # https
    acl Safe_ports port 70          # gopher
    acl Safe_ports port 210         # wais
    acl Safe_ports port 1025-65535  # unregistered ports
    acl Safe_ports port 280         # http-mgmt
    acl Safe_ports port 488         # gss-http
    acl Safe_ports port 591         # filemaker
    acl Safe_ports port 777         # multiling http
    acl CONNECT method CONNECT#
    # Recommended minimum Access Permission configuration:
    #
    # Deny requests to certain unsafe ports
    http_access deny !Safe_ports# Deny CONNECT to other than secure SSL ports
    http_access deny CONNECT !SSL_ports# Only allow cachemgr access from localhost
    http_access allow localhost manager
    http_access deny manager# We strongly recommend the following be uncommented to protect innocent
    # web applications running on the proxy server who think the only
    # one who can access services on "localhost" is a local user
    #http_access deny to_localhost#
    # INSERT YOUR OWN RULE(S) HERE TO ALLOW ACCESS FROM YOUR CLIENTS
    ## Example rule allowing access from your local networks.
    # Adapt localnet in the ACL section to list your (internal) IP networks
    # from where browsing should be allowed
    http_access allow localnet
    http_access allow localhost# And finally deny all other access to this proxy
    http_access deny all# Squid normally listens to port 3128
    http_port 3128# Uncomment and adjust the following to add a disk cache directory.
    #cache_dir ufs /var/spool/squid 100 16 256# Leave coredumps in the first cache dir
    coredump_dir /var/spool/squid

# # Add any of your own refresh_pattern entries above these.
refresh_pattern ^ftp:           1440    20%     10080
refresh_pattern ^gopher:        1440    0%	1440
refresh_pattern -i (/cgi-bin/|\?) 0     0%	0
refresh_pattern .               0       20%     4320
```

### Allow IP Address to Use the Internet Through Your Proxy Server

If your netwok is 110.220.330.0/24:
```
acl localnet src 110.220.330.0/24
```

For changes to take effect you will need to restart your Squid server, use the following command for same.

```
systemctl restart squid
```

### Allow a Specific Port for HTTP Connections

```
acl Safe_ports port 8080
```

### Using Basic Authentication with Squid

```
yum -y install httpd-tools
touch /etc/squid/passwd && chown squid /etc/squid/passwd

htpasswd /etc/squid/passwd proxyuser
    New password:
    Re-type new password:
    Adding password for user pxuser
```

```
nano /etc/squid/squid.conf
...
auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic children 5
auth_param basic realm Squid Basic Authentication
auth_param basic credentialsttl 2 hours
acl auth_users proxy_auth REQUIRED
http_access allow auth_users
```

### Blocking Websites

```
nano /etc/squid/blocked_sites
facebook.com
youtube.com
```

```
nano /etc/squid/squid.conf
acl blocked_sites dstdomain "/etc/squid/blocked_sites"
http_access deny blocked_sites
```

### Block Specific Keyword with Squid
```
nano /etc/squid/blockkeywords.lst
yahoo
gmail
facebook
```

```
acl blockkeywordlist url_regex "/etc/squid/blockkeywords.lst"
http_access deny blockkeywordlist
```

### Disable caching
```
# Leave coredumps in the first cache dir
#coredump_dir /var/spool/squid
```

### Changing Squid Port
```
nano /etc/squid/squid.conf
http_port 3128
```

### Export Proxy Server Settings

```
$ export http_proxy="http://PROXY_SERVER:PORT"
$ export https_proxy="http://PROXY_SERVER:PORT"
$ export ftp_proxy="http://PROXY_SERVER:PORT"

$ export http_proxy="http://USER:PASSWORD@PROXY_SERVER:PORT"
$ export https_proxy="http://USER:PASSWORD@PROXY_SERVER:PORT"
$ export ftp_proxy="http://USER:PASSWORD@PROXY_SERVER:PORT"
```

### Test caching

```
env | grep proxy

tailf /var/log/squid/access.log
```

