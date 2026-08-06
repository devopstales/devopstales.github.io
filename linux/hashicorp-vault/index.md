---
title: Install hashicorp vault
url: https://devopstales.github.io/linux/hashicorp-vault/
date: 2019-05-17
keywords: HashiCorp Vault, Centos
---


Hashicorp vault is a highly scalable, highly available, environment agnostic way to generate, manage, and store secrets.
<!--more-->

### Dowload  Vault
```
# https://releases.hashicorp.com/vault/
cd /opt
VAULT_VERSION="1.1.2"
curl -sO https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip

unzip vault_${VAULT_VERSION}_linux_amd64.zip
mv vault /usr/bin/
vault --version
```

```
vault -autocomplete-install
complete -C /usr/bin/vault vault
mkdir /etc/vault
mkdir -p /var/lib/vault/data

sudo useradd --system --home /etc/vault --shell /bin/false vault
sudo chown -R vault:vault /etc/vault /var/lib/vault/
```

### Configure Vault systemd service
```
nano /etc/systemd/system/vault.service
[Unit]
Description=vault server
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault/config.hcl

[Service]
User=vault
Group=vault
Restart=on-failure
ExecStart=/usr/bin/vault server -config=/etc/vault
ExecStop=/usr/bin/vault step-down
#ExecReload=/bin/kill --signal HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

### Create vault config
```
nano /etc/vault/config.hcl
disable_cache = true
disable_mlock = true
ui = true
listener "tcp" {
   address          = "0.0.0.0:8200"
   tls_disable      = 1
}
storage "file" {
   path  = "/var/lib/vault/data"
 }
api_addr         = "http://0.0.0.0:8200"
max_lease_ttl         = "10h"
default_lease_ttl    = "10h"
cluster_name         = "vault"
raw_storage_endpoint     = true
disable_sealwrap     = true
disable_printable_check = true
```

```
systemctl daemon-reload
systemctl enable --now vault
systemctl status vault
```

### Configurate Client
```
export VAULT_ADDR=http://127.0.0.1:8200
echo "export VAULT_ADDR=http://127.0.0.1:8200" >> ~/.bashrc

sudo rm -rf  /var/lib/vault/data/*
vault operator init > /etc/vault/init.file

cat /etc/vault/init.file | grep "Initial Root Token:"
export VAULT_TOKEN="s.RcW0LuNIyCoTLWxrDPtUDkCw"

### go to gou and un seal with 3 keys

vault status
```

### Configurate user-base authentication
```
vault auth enable userpass
vault write auth/userpass/users/devopstales \
    password=Password1 \
    policies=admins
```

