---
title: Migrate BIND to Windows DNS
url: https://devopstales.github.io/windows/migrate-bind-to-windows-dns/
date: 2019-06-04
---


Migrate dns Zones from bind to Windows DNS Server
<!--more-->
### Linux

```
nano /etc/bind/zones.active
allow-transfer { 192.168.0.8; };
```

### Windows

```
Add-DnsServerSecondaryZone -Name "devopstales.intra" -ZoneFile "devopstales.intra.dns" -MasterServers 192.168.0.60
ConvertTo-DnsServerPrimaryZone -Name "devopstales.intra" -PassThru -Verbose -ZoneFile "devopstales.intra.dns" -Force
Set-DnsServerPrimaryZone -Name "devopstales.intra" –Notify Notifyservers –notifyservers "192.168.0.5" -SecondaryServers "192.168.0.5" –SecureSecondaries TransferToSecureServers
```

