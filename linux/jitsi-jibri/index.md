---
title: Install recording fot Jitsi
url: https://devopstales.github.io/linux/jitsi-jibri/
date: 2020-10-21
keywords: jitsi
---


In this post I will show you my productivity tips with kubectlyo can install Jibri the recording component of Jitsi.
<!--more-->

### Install requirements
```
sudo apt install ffmpeg curl unzip software-properties-common
```

Enable the ALSA loopback module to start on boot,
```
echo "snd_aloop" >> /etc/modules
modprobe snd_aloop
```

### Install Google Chrome:
```
curl -o - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add
echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google-chrome.list
apt update
apt install google-chrome-stable

# install headless browser driver
CHROME_DRIVER_VERSION=`curl -sS chromedriver.storage.googleapis.com/LATEST_RELEASE`
wget -N http://chromedriver.storage.googleapis.com/$CHROME_DRIVER_VERSION/chromedriver_linux64.zip -P ~/
unzip ~/chromedriver_linux64.zip -d ~/
rm ~/chromedriver_linux64.zip
sudo mv -f ~/chromedriver /usr/local/bin/chromedriver
sudo chmod 0755 /usr/local/bin/chromedriver

# disable warnings
mkdir -p /etc/opt/chrome/policies/managed
echo '{ "CommandLineFlagSecurityWarningsEnabled": false }' >>/etc/opt/chrome/policies/managed/managed_policies.json
```

### Install jibri
```
apt install jibri
usermod -aG adm,audio,video,plugdev jibri

nano /etc/prosody/prosody.cfg.lua
...
Component "conference.jitsi.mydomain.intra" "muc"
modules_enabled = { "muc_mam" }
...
Component "internal.auth.jitsi.mydomain.intra" "muc"
modules_enabled = {
"ping";
}
storage = "internal"
muc_room_cache_size = 1000
...
VirtualHost "recorder.jitsi.mydomain.intra"
modules_enabled = {
"ping";
}
authentication = "internal_plain"
```

Create accounts:
```
prosodyctl register jibri auth.jitsi.mydomain.intra Password1
prosodyctl register recorder recorder.jitsi.mydomain.intra Password1
```

```
sudo nano /etc/jitsi/jicofo/sip-communicator.properties
...
org.jitsi.jicofo.jibri.BREWERY=JibriBrewery@internal.auth.jitsi.mydomain.intra
org.jitsi.jicofo.jibri.PENDING_TIMEOUT=90

sudo nano /etc/jitsi/meet/jitsi.mydomain.intra-config.js
...
fileRecordingsEnabled: true,
liveStreamingEnabled: true,
hiddenDomain: 'recorder.jitsi.mydomain.intra',
```

Configure storage:
```
mkdir /recordings
chown jibri:jibri /recordings

sudo nano /etc/jitsi/jibri/config.json
"recording_directory": "/recordings",
"finalize_recording_script_path": "",
"xmpp_server_hosts": [
"jitsi.mydomain.intra"
],
"xmpp_domain": "jitsi.mydomain.intra",
"control_login": {
"domain": "auth.jitsi.mydomain.intra",
"username": "jibri",
"password": "Password1d"
},
"control_muc": {
"domain": "internal.auth.jitsi.mydomain.intra",
"room_name": "JibriBrewery",
"nickname": "jibri"
},
"call_login": {
"domain": "recorder.jitsi.mydomain.intra",
"username": "recorder",
"password": "Password1"
},
```

### Install Java 8
Jibri only works with java 1.8 so we need to install and configure Jibri to use it:
```
wget -O - https://adoptopenjdk.jfrog.io/adoptopenjdk/api/gpg/key/public | sudo apt-key add -
add-apt-repository https://adoptopenjdk.jfrog.io/adoptopenjdk/deb/
apt update
apt install adoptopenjdk-8-hotspot

# the systemd sript use this script so we edit it
nano /opt/jitsi/jibri/launch.sh
# change this:
# exec java ...
# tho this:
exec /usr/lib/jvm/adoptopenjdk-8-hotspot-amd64/bin/java ...
```

```
systemctl restart prosody jicofo jitsi-videobridge2 jibri
systemctl enable jibri
```

