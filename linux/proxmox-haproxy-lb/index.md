---
title: Proxmox: Behind HAproxy Load Balancer
url: https://devopstales.github.io/linux/proxmox-haproxy-lb/
date: 2023-10-05
keywords: proxmox ve, proxmox cluster, certificate, Proxmox Backup Server, proxmox ve, PBS, HAproxy, pfSense
---


In thist post I will show you how to configure HAproxy to Load Balancer between the Proxmox VE Web interfaces and get a single hostname for your cluster.

<!--more-->

I have multiple proxmox clusters with different hostname patterns so find the right hostname for a specific cluster can be changing. I decided to use the HAproxy on pfSense to Load Balancer between the Proxmox VE Web interfaces. I believed it vhould be easy... Well not so easy.

### DNS

This first task is to Create a DNS record pointing to the pfSense.

### HAproxy backend

The nex part is to create a HAproxy backend on pfSense. To do this go to `Services > HAProxy > Backend` and select `Add`.

At the `Server list` add all your servers with port 8006 (port of the web ui) then enable `Encrypt(SSL)` and `SSL checks`. If you need you can add the your ssl CA here.

![HAProxy backend](/img/include/proxmox_haproxy_be01.webp)

At `Load balancing options` select `Round robin`.

![HAProxy backend](/img/include/proxmox_haproxy_be02.webp)

Increase the timeout.

![HAProxy backend](/img/include/proxmox_haproxy_be03.webp)

Set the `Health check method` to `Basic`.

![HAProxy backend](/img/include/proxmox_haproxy_be04.webp)

> Here comes the most important part. First i didn't configured Cookie persistence nor Sticky sessions. Without this option the proxmox webvnc dose not work properly.

![HAProxy backend](/img/include/proxmox_haproxy_be05.webp)

Then set Sticky sessions.

![HAProxy backend](/img/include/proxmox_haproxy_be06.webp)

### HAproxy Frontend

The next part is to crete the frontend.  Select `frontend` at the top menu of the pfService HAProxy menu. Then configure the `Access Control List` to match your chosen hostname.

![HAProxy frontend](/img/include/proxmox_haproxy_fe01.webp)

At the `Actions` part select the ACL you created in the previous config. Then select your backend.

![HAProxy frontend](/img/include/proxmox_haproxy_fe02.webp)

To configure HAProxy to serv this frontent on https you need to configure SSL ofloading. Tick the option `Use Ofloading` and select the certificate you want to use for this site. For this demo I created a Let's encrypt certificate.

![HAProxy frontend](/img/include/proxmox_haproxy_fe03.webp)

