---
title: Configure squid transparent proxy on pfsense
url: https://devopstales.github.io/linux/pfsense-suidproxy/
date: 2022-11-12
keywords: pfsense, squid, transparent proxy
---


In this post I will show you how you can install squid proxy on pfsense and configure as a transparent proxy.
<!--more-->

### Install Squid Package on pfSense

Go to the **System > Package Manager** and search to `squid`.

![Package Manager](/img/include/pfSense_squid001.webp)

Then install `squid` and `SquidGuard` package:

![Install package](/img/include/pfSense_squid002.webp)
### Configuring Squid Proxy Server on pfSense

Go to **Services > Squid Proxy Server** To enable the Squid Proxy we have to check **Enable Squid Proxy**.

Here you can select under **Proxy Interface(s)**, the interface which the proxy server should listen and bind to. Also be sure that Allow Users on Interface is checked. If this is checked, the subnets for the interfaces selected will automatically have access. There will be no need to add them on the **Access Control Lists (ACLs)** tab.

![squid settings](/img/include/pfSense_squid007.webp)

If you enable **Transparent HTTP Proxy** the clients do not need any additional configuration like environment variables or proxy settings in the browser to use the forward proxy.

![Http transparent proxy](/img/include/pfSense_squid003.webp)

By default Transparent HTTP Proxy only forwards requests for destination port 80. In order to proxy HTTPS the proxy should know the requested host and port number which will be encrypted with POST and GET requests with transparent proxy. Therefore you should enable **intercepting SSL connections** or configure **WPAD/PAC** option on the DNS/DHCP server in order to let the client send CONNECT requests.

### Access Control Lists (ACLs)

In the **ACLs tab** for now we only configured above our allowed subnets who can access and request outbound internet access.

![ACL](/img/include/pfSense_squid005.webp)
### Configure Squid Proxy Logging Settings

Default Logging is not enabled. If you want to enable Access Logging go to Logging Settings under the General menu tab.

![logging config](/img/include/pfSense_squid004.webp)

Under the Real Time tab you can see the latest access logs regarding requested destinations from the clients.

![real time logs](/img/include/pfSense_squid006.webp)

