---
title: Install Errata
url: https://devopstales.github.io/linux/katello-errata/
date: 2019-05-02
---


Katello brings the full power of content management alongside the provisioning and configuration capabilities of Foreman. Katello is the upstream community project from which the Red Hat Satellite product is derived after Red Hat Satellite Server 6.
<!--more-->


### Errata
```
yum install -y git \
pulp-admin-client \
pulp-rpm-admin-extensions \
pulp-rpm-consumer-extensions \
pulp-rpm-handlers \
pulp-rpm-yumplugins \
pulp-rpm-admin-extensions \
pulp-consumer-client \
python-pulp-agent-lib \
perl-Text-Unidecode \
perl-XML-Simple \
perl-XML-Parser

cd /opt
git clone https://github.com/rdrgmnzs/pulp_centos_errata_import.git
cd ./pulp_centos_errata_import
wget -N https://cefs.steve-meier.de/errata.latest.xml.bz2
bunzip2 ./errata.latest.xml.bz2
mkdir -m0700 ~/.pulp
cat /etc/pki/katello/certs/pulp-client.crt /etc/pki/katello/private/pulp-client.key > ~/.pulp/user-cert.pem
chmod 0400 ~/.pulp/user-cert.pem

# import Errata
perl ./errata_import.pl --errata=errata.latest.xml

pulp-admin repo list | less

add rarrata to rebo by repoid

perl ./errata_import.pl \
--errata=errata.latest.xml \
--include-repo=6356cfe9-766e-4f07-a85e-f287b25395e3 \
--include-repo=2-el7_content-v1_0-6356cfe9-766e-4f07-a85e-f287b25395e3 \
--include-repo=2-el7_content-Library-6356cfe9-766e-4f07-a85e-f287b25395e3 \
--include-repo=2-el7_content-stable-6356cfe9-766e-4f07-a85e-f287b25395e3 \
--include-repo=cb299646-b1a3-4225-9c78-986851a1725f \
--include-repo=2-el7_content-v1_0-cb299646-b1a3-4225-9c78-986851a1725f \
--include-repo=2-el7_content-Library-cb299646-b1a3-4225-9c78-986851a1725f \
--include-repo=2-el7_content-stable-cb299646-b1a3-4225-9c78-986851a1725f
```

### Update repo
```
hammer repository synchronize \
--skip-metadata-check true \
--name "base_x86_64" \
--product "el7_repos"

hammer repository synchronize \
--skip-metadata-check true \
--name "updates_x86_64" \
--product "el7_repos"

hammer content-view publish \
--name "el7_content" \
--description "Publishing repositories"

hammer content-view version promote \
--content-view "el7_content" \
--version "4.0" \
--to-lifecycle-environment "stable"
```

