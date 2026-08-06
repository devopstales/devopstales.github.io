---
title: Subject Alternative Name in Active Dyrectory LDAPS Cerificate
url: https://devopstales.github.io/windows/msad-ldaps-subject-alternative-mame/
date: 2021-07-22
---


In this post I will show you how you can configure custom Subject Alternative Name in Active Directory LDAPS certificate.
<!--more-->

### Open mmc

* `windows + r`
* run `mmc`

![Example image](/img/include/ldapssan1.webp)

* Click `File / Add/Remove Snap-in..` or `ctrl + m`

![Example image](/img/include/ldapssan2.webp)
![Example image](/img/include/ldapssan3.webp)

* Add certificates

![Example image](/img/include/ldapssan4.webp)
![Example image](/img/include/ldapssan5.webp)

* Add a nother certificates for service

![Example image](/img/include/ldapssan6.webp)
![Example image](/img/include/ldapssan5.webp)
![Example image](/img/include/ldapssan7.webp)

* Add Certificate Authoraty

![Example image](/img/include/ldapssan8.webp)
![Example image](/img/include/ldapssan5.webp)

### Clone Template
* Right click on `Certificate Authoraty / CA NAME / Certificate Template` and select `Manage`

![Example image](/img/include/ldapssan10.webp)

* Select `Domain Controller Template`
* Right Click and `Duplicate template`

![Example image](/img/include/ldapssan11.webp)
![Example image](/img/include/ldapssan12.webp)
![Example image](/img/include/ldapssan22.webp)

* Then click `OK` and close the `Certificate Teplate Console`

### Add template to Certificate Template list
* At `Certificate Authoraty / Domain Controller / Certificate Template`

![Example image](/img/include/ldapssan9.webp)

* Rght click on `Certificate Template` and select `New / Certificate Template to Issue` Add the new Template

![Example image](/img/include/ldapssan13.webp)

### Generate Certificate
* Right click on `Certificates (Local Computer) / Personal / Certificate` and select `All Tasks / Request New Certificate`

![Example image](/img/include/ldapssan16.webp)
![Example image](/img/include/ldapssan17.webp)
![Example image](/img/include/ldapssan18.webp)
![Example image](/img/include/ldapssan19.webp)
![Example image](/img/include/ldapssan20.webp)

* enroll

![Example image](/img/include/ldapssan21.webp)


#### Change Certificate

* To activate the new certificate you need to restart the Domain Controller

