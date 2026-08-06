---
title: Google Authenticator on pfSense
url: https://devopstales.github.io/linux/pfsense-2fa/
date: 2019-04-29
keywords: pfsense, 2FA, radius, firewall
---


This article explains how to set up OpenVPN with Google Authenticator on pfSense.
<!--more-->

### Set up the FreeRADIUS

* Go to  `System > Package Manager > Available Packages` and install `FreeRADIUS` package.
*  `Services > FreeRADIUS > Interfaces > Add`

|   |   |
|---|---|
| Interface IP Address | 127.0.0.1 |
| Port | 1812 |
| Interface Type | Authentication |
| IP Version | IPv4 |
| Description | Authentication |

![Example image](/img/include/pfsense_2fa_1.webp)

|   |   |
|---|---|
| Interface IP Address | 127.0.0.1 |
| Port | 1813 |
| Interface Type | Authentication |
| IP Version | IPv4 |
| Description | Accounting |

![Example image](/img/include/pfsense_2fa_2.webp)

### Add a NAS client

* `Services > FreeRADIUS > NAS/Clients > Add`

|   |   |
|---|---|
| Client IP Address | 127.0.0.1 |
| Client IP Version | IPv4 |
| Client Shortname | pfsenselocal |
| Client Shared Secret | Password1 |
| Client Protocol | UDP |
| Client Type | other |
| Require Message Authenticator | No |
| Max Connections | 16 |
| Description | pfsenselocal |

![Example image](/img/include/pfsense_2fa_3.webp)

### Add an authentication server ro pfSense

* `System > User Manager > Authentication Servers > Add`

|   |   |
|---|---|
| Descriptive Name | localfreeradius |
| Type | RADIUS |
| Protocol | PAP |
| Hostname or IP address | 127.0.0.1 |
| Shared Secret | Password1 |
| Services offered | Authentication and Accounting |
| Authentiocation port | 1812 |
| Accounting port | 1813 |
| Authentication Timeout | 5 |
| RADIUS NAS IP Attribute | LAN |

![Example image](/img/include/pfsense_2fa_4.webp)

### Configurate OTP for Users

- `Services > FreeRADIUS > Users > Add`

|   |   |
|---|---|
| Username | tester |
| Password |   |
| Password Encryption | Cleartext-Password |
| One-Time Password | Enable One-Time Password (OTP) for this user |
| OTP Auth Method | Google-Authenticator |
| Init-Secret | click Generator OTP Secret |
| PIN | enter 4-8 numbers and remember them. |
| QR Code | click Generate QR Code. |

At this point open Google Authenticator on your phone and scan the QRCODE.

![Example image](/img/include/pfsense_2fa_5.webp)

You can use One-Time Password (OTP) only for local FreeRadius users. FreeRadius users from diferent backenl like mysql or ldap did not work.

### Configurate openvpn

* Go to `VPN > OpenVPN > Servers > Edit`
* Select localfreeradius for Backend for authentication

![Example image](/img/include/pfsense_2fa_6.webp)

* In the OpenVPN Server configuration, under `Advanced Configuration > Custom options`
* add: `reneg-sec 0`

If you connect your OpenVPN client you must enter your username and the PIN + the Google Authenticator one-time code as your password. <br> If PIN is 1234 and the Google Authenticator code is 445 745 then the password is: 1234445745

