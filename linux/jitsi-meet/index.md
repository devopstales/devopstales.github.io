---
title: Install Jitsi meet
url: https://devopstales.github.io/linux/jitsi-meet/
date: 2020-10-20
keywords: jitsi, recording, streaming
---


In this post I will show you how you can install Jitsi meet on your server.
<!--more-->

Jitsi is state-of-the art video conferencing software that you can self-host or simply use at meet.jit.si.

### Configure hostname
In this step, you will change the system’s hostname to match the domain name that you intend to use for your Jitsi Meet instance.
```
sudo hostnamectl set-hostname jitsi.mydomain.intra
sudo nano nano /etc/hosts
127.0.0.1 jitsi.mydomain.intra
```

### Install Jitsi Meet
```
curl https://download.jitsi.org/jitsi-key.gpg.key | \
sudo sh -c 'gpg --dearmor > /usr/share/keyrings/jitsi-keyring.gpg'

echo 'deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/' | \
sudo tee /etc/apt/sources.list.d/jitsi-stable.list > /dev/null
```

```
sudo apt install apt-transport-https
sudo apt update
sudo apt install jitsi-meet
```
During the installation of `jitsi-meet` you will be prompted to enter the domain name. Then you will be shown a new dialog box that asks if you want Jitsi to create and use a self-signed TLS certificate or use an existing one. For this demo I will use the self-signed option.

If you want to use letsencrypt select self-signed the install certboot and use jitsi's script like this:
```
sudo add-apt-repository ppa:certbot/certbot
sudo apt install certbot
sudo /usr/share/jitsi-meet/scripts/install-letsencrypt-cert.sh
```

### NAT Configuration
If the installation is behind NAT jitsi-videobridge should configure jitsi-videobridge  in order for it to be accessible from outside.
```
nano /etc/jitsi/videobridge/sip-communicator.properties
org.ice4j.ice.harvest.NAT_HARVESTER_LOCAL_ADDRESS=<Local.IP.Address>
org.ice4j.ice.harvest.NAT_HARVESTER_PUBLIC_ADDRESS=<Public.IP.Address>
```
The you need to NAT the ports 443 4443 10000 from externat ip to the server.

### Configure Jitsi
The default config allow any user to start meetings without authentication. For a publicly accessible server this is not what we want.

With this configuration all users need to authenticate to use Jitsi.
```
sudo nano /etc/prosody/conf.avail/jitsi.mydomain.intra.cfg.lua
...
VirtualHost "jitsi.mydomain.intra"
# change this:
#       authentication = "anonymous"
# tho this:
        authentication = "internal_plain"

sudo nano /etc/jitsi/jicofo/sip-communicator.properties
...
org.jitsi.jicofo.auth.URL=XMPP:jitsi.mydomain.intra
```

Create users manually
```
prosodyctl register <username> jitsi.mydomain.intra <Password>
```
For a few user is is ok to create them manually but for many users you need something like ldap:

```
sudo apt install sasl2-bin libsasl2-modules-ldap lua-cyrussasl
sudo nano /etc/prosody/conf.avail/ldap.cfg.lua
VirtualHost "jitsi.mydomain.intra"
# change this:
#       authentication = "anonymous"
# tho this:
        authentication = "cyrus"
        cyrus_application_name = "xmpp"
...
        modules_enabled = {
...
            "auth_cyrus"; -- Add this line
        }

        c2s_require_encryption = false
...
```

```
sudo nano /etc/sasl/xmpp.conf
pwcheck_method: saslauthd
mech_list: PLAIN

sudo nano /etc/saslauthd.conf
ldap_servers: ldap://10.0.0.1
ldap_search_base: dc=my,dc=search,dc=base
ldap_bind_dn: cn=Administrator,cn=Users,dc=foo,dc=bar
ldap_bind_pw: PassW0rd
ldap_filter: (samaccountname=%u)
ldap_version: 3
ldap_auth_method: bind
# for tls change ldap_servers to ldaps://
# the add uncomment and configure this:
#ldap_tls_key: /config/certs/meet.jit.si.key
#ldap_tls_cert: /config/certs/meet.jit.si.crt

#ldap_tls_check_peer: yes
#ldap_tls_cacert_file: /etc/ssl/certs/ca-certificates.crt
#ldap_tls_cacert_dir: /etc/ssl/certs
```

The next configuration allows anonymous users to join conference rooms that were created by an authenticated user.
```
sudo nano /etc/prosody/conf.avail/jitsi.mydomain.intra.cfg.lua
...
# to the end
VirtualHost "guest.jitsi.mydomain.intra"
    authentication = "anonymous"
    c2s_require_encryption = false

```
The `guest.` hostname is only used internally by Jitsi Meet. You will never enter it into a browser or need to create a DNS record for it.

```
sudo nano /etc/jitsi/meet/jitsi.mydomain.intra-config.js
var config = {
    hosts: {
        domain: "jitsi.mydomain.intra",
        anonymousdomain: 'guest.jitsi.mydomain.intra',
```

After a config change yo need to restart the services:
```
systemctl restart prosody jicofo jitsi-videobridge2
```

More info:
---
* https://github.com/jitsi/jitsi-meet/wiki/LDAP-Authentication#configure-saslauthd

