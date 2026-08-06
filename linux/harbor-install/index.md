---
title: Install vMWare Harbor
url: https://devopstales.github.io/linux/harbor-install/
date: 2019-04-18
---


Vmware harbor ia an open source trusted cloud native registry project that stores, signs, and scans content.
<!--more-->

Why harbor? Opeshift and Gitlab has its own docker regytry but nether can intgrate with clair Vulnerability scanner.

### Install Docker and Docker-Compose
```
yum install epel-release wget -y
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce

sudo yum install -y python-pip
pip install docker-compose

sudo systemctl start docker
sudo systemctl enable docker
```

### Generate your own SSL certificate
```
nano certgen.sh
#!/bin/sh

export PASSPHRASE=$(head -c 500 /dev/urandom | tr -dc a-z0-9A-Z | head -c 128; echo)
DOMAIN=devopstales.intra

subj="
C=HU
ST=Pest
O=My Company
localityName=Budapest
commonName=*.$DOMAIN
organizationalUnitName=OU
emailAddress=root@$DOMAIN
"

openssl genrsa -des3 -out domain.key -passout env:PASSPHRASE 2048

openssl req \
    -new \
    -batch \
    -subj "$(echo -n "$subj" | tr "\n" "/")" \
    -key domain.key \
    -out domain.csr \
    -passin env:PASSPHRASE

cp domain.key domain.key.org

openssl rsa -in domain.key.org -out domain.key -passin env:PASSPHRASE

openssl x509 -req -days 3650 -in domain.csr -signkey domain.key -out domain.crt
cat domain.crt domain.key > domain.pem
```

```
chmod +x certgen.sh
./certgen.sh

mkdir -p /etc/docker/certs.d/harbor.devopstales.intra
cp domain.crt domain.key /etc/docker/certs.d/harbor.devopstales.intra/
cp domain.crt /etc/docker/certs.d/harbor.devopstales.intra/domain.cert
sudo systemctl restart docker
```

### Install notary
```
curl -L https://github.com/theupdateframework/notary/releases/download/v0.6.1/notary-$(uname -s)-amd64 -o /usr/local/bin/notary
chmod +x /usr/local/bin/notary

mkdir -p ~/.docker/tls/harbor.devopstales.intra:4443/
cp ~/domain.crt ~/.docker/tls/harbor.devopstales.intra:4443/
cp ~/domain.key ~/.docker/tls/harbor.devopstales.intra:4443/
cp ~/domain.crt ~/.docker/tls/harbor.devopstales.intra:4443/domain.cert
```

### Install Harbor
```
# https://github.com/vmware/harbor/releases/
wget https://storage.googleapis.com/harbor-releases/release-1.7.0/harbor-online-installer-v1.7.5.tgz
tar -xzf harbor-online-installer-v1.7.5.tgz

cd harbor
nano harbor.cfg
hostname = harbor.devopstales.intra
ui_url_protocol = https
ssl_cert = /root/domain.crt
ssl_cert_key = /root/domain.key

./prepare
./install.sh --with-notary --with-clair

docker login harbor.devopstales.intra
```

Access the Harbor UI with the username "admin" and password "Harbor12345"
![Example image](/img/include/harbor_1.webp)

Create a nwe project.
![Example image](/img/include/harbor_2.webp)

Configure automatic Vulnerability scan for project.
![Example image](/img/include/harbor_3.webp)

```
docker pull nginx
docker tag nginx:latest harbor.devopstales.intra/test/nginx:V1
docker push harbor.devopstales.intra/test/nginx:V1
```

![Example image](/img/include/harbor_4.webp)

```
docker tag nginx:latest harbor.devopstales.intra/test/nginx:V2
export DOCKER_CONTENT_TRUST_SERVER=https://harbor.devopstales.intra:4443
export DOCKER_CONTENT_TRUST=1
docker push harbor.devopstales.intra/test/nginx:V2
```

![Example image](/img/include/harbor_5.webp)

