---
title: Configure opnsense nextcloud backup
url: https://devopstales.github.io/linux/opnsense-nextcloud/
date: 2019-05-25
---


In this LAB I will be configurate the opnsense cloud backup solutuon for nextcloud.
<!--more-->

### Configurate the nextcloud
- login to your Nextcloud instance with the admin account
- go to users
![Example image](/img/include/nextcloud_sso1.webp)

- create a new user for opnsense
![Example image](/img/include/opnsense_nextcloud2.webp)

- login with tehe new user and go to `profiele > settings > seurity`
- create token for user
![Example image](/img/include/opnsense_nextcloud3.webp)



### Configurate opnsense backup

- login to opnsense
- go to `system > config > bckups`
- enable the nextclod config
![Example image](/img/include/opnsense_nextcloud4.webp)
- click the `setup/test nextcloud` button

