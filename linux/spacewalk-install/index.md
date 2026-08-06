---
title: Install spacewalk 2.9
url: https://devopstales.github.io/linux/spacewalk-install/
date: 2019-04-29
---


Spacewalk is an open source Linux systems management solution. Spacewalk is the upstream community project from which the Red Hat Satellite product is derived before Red Hat Satellite Server 6.
<!--more-->

### Create answerfile
```
cat > /root/spacewalk_answers.txt << EOF
admin-email = operation@devopstales.intra
ssl-set-cnames = spacewalk
ssl-set-org = Spacewalk Org
ssl-set-org-unit = spacewalk
ssl-set-city = Budapest
ssl-set-state = non
ssl-set-country = HU
ssl-password = Password1
ssl-set-email = operation@devopstales.intra
ssl-config-sslvhost = Y
enable-tftp=Y
EOF
```

### Install requirements
```
yum install ntp -y
service ntpd restart


rpm -Uvh https://copr-be.cloud.fedoraproject.org/results/@spacewalkproject/spacewalk-2.9/epel-7-x86_64/00830557-spacewalk-repo/spacewalk-repo-2.9-4.el7.noarch.rpm

rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

yum clean all && yum repolist
```

### Install Spacewalk
```
yum install -y spacewalk-setup-postgresql
yum install -y spacewalk-postgresql
spacewalk-setup --answer-file=/root/spacewalk_answers.txt
yum install spacecmd -y
```

Go to the WebUI and configure the admin user.

