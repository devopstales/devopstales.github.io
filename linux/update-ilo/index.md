---
title: Update ILO firmware
url: https://devopstales.github.io/linux/update-ilo/
date: 2019-04-29
---


Integrated Lights-Out, or iLO, is a proprietary embedded server management technology by Hewlett-Packard which provides out-of-band management facilities.
<!--more-->

### Update from Linux CLI
Download iLO firmware say CP032487.scexe
```
cd /tmp
chnmod 755 CP032487.scexe
./CP032487.scexe
curl http:///xmldata?item=All
```

### Update from ILO CLI
```
./CP032487.scexe --unpack=firmware
cd firmware
cp firmware/ilo2_NNN.bin $DocumentRoot (usually /var/www/html/)

# Login on iLO over ssh and upload firmware
ssh -l Administrator

iLO> show
iLO> cd /map1
iLO> oemhp_ping
iLO> load -source http:///firmware/ilo2_NNN.bin

curl http:///xmldata?item=All
```

### Update from iLO WebUI
- access iLO Web UI, go to Administration tab and upload bin file to upgrade iLO firmware

