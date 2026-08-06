---
title: How to: Configure Firefox to use Windows Certificate Store via GPO
url: https://devopstales.github.io/windows/configure-firefox-to-use-windows-certificate-store-via-gpo/
date: 2022-11-16
keywords: pfsense, squid, transparent proxy, Active Directory, Group Policy , firefox, Windows Certificate Store, GPO, AD
---


I ran into an issue when I enabled HTTPS Inspection on our transparent proxy and Firefox had a certificate error for everyone.
<!--more-->

### Distribute CA Certificate with GPO

You need to place the certificate file to the shared network folder and all users must have a read access to it. Then start the **Group Policy Management console (gpmc.msc).** Select an OU then right click and select **Create a GPO in this domain and Link it here…**

![create gpo](/img/include/ssl-gpo001.webp)

In the GPO Editor, go to the section **Computer Configuration -> Policies -> Windows Settings -> Security Settings -> Public Key Policies -> Trusted Root Certification Authorities.**

Right-click in the right part of the GPO editor window and select Import.

![import cert to gpo](/img/include/ssl-gpo002.webp)

Specify the path to the imported certificate file, which you have placed in the shared folder.

In the corresponding step of the wizard (Place all certificates in the following store), do specify that it has to be placed in the **Trusted Root Certification Authorities.**

![import cert to gpo](/img/include/ssl-gpo003.webp)

The certificate distribution policy created. Let’s test the group policy settings  by running `gpupdate /force` on the client.  Verify that your certificate has appeared in the list of trusted certificates. In the Internet Explorer settings (**Internet Options -> Content -> Certificates -> Trusted Root Certification Authorities**).

### Configure Firefox to use Windows Certificate Store via GPO

The previous GPO did not solved all my problem, but by default Firefox does not look at the Windows Certificate Store. So I looked around on the inter nat. Since Firefox 49 there is some support for Windows CA certificates and support for Active Directory provided enterprise root certificates since Firefox 52. It is also supported in macOS to read from the Keychain since version 63.

Since Firefox 68 this feature is enabled by default in the ESR (enterprise) version, but not in the (standard) rapid release.

When Firefox opens, it runs any .js scripts in the following location:

```bash
C:\Program Files (x86)\Mozilla Firefox\Defaults\Pref\ - 64 Bit Machine 
C:\Program Files\Mozilla Firefox\Defaults\Pref\ - 32 Bit Machine
```

You will need to create a file called `Enableroot.js` (or similar) with the following contents:

```js
/* Allows Firefox reading Windows certificates */
pref(security.enterprise_roots.enabled, true);
```

First place the js in the same shared folder then create a new GPO. I had already created a GPO to deploy a CA cert across our domain, so I just edited this one. 

Edit GPO, and navigate to: **Computer Config -> Preferences -> Windows Settings -> Files**

Right click, select New then File. 

* Set the Action to Create. 
* In Source File type the UNC path to the shared `Enableroot.js` mentioned before.
* In Destination file you want one of the following: `C:\Program Files (x86)\Mozilla Firefox\Defaults\Pref\enableroot.js`

> If you have both 32 and 64 bit machines you can make a copy of this with the festination file path `C:\Program Files\Mozilla Firefox\Defaults\Pref\enableroot.js`

