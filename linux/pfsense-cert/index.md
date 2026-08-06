---
title: Pfsese https
url: https://devopstales.github.io/linux/pfsense-cert/
date: 2019-04-15
---


I will show you how to Enable SSL for pfSense.
<!--more-->

### Creating a new Certificate
At `System > Certificate Manager > Certificates > Add`<br>
Make sure you choose "Import an existing Certificate" under Method and enter Descriptive name so you know what the certificate is.
![Example image](/img/include/pfsense_cert_1.webp)

At System > Advanced > Admin Access<br>
Make sure HTTPS is selected as Protocol and now change the SSL Certificate to the one you have created. Scroll down and click on Save. Now, when you restart your Web Browser, you should see a Secure Connection to pfSense when accessing it.
![Example image](/img/include/pfsense_cert_2.webp)

