---
title: Store docker credentials in keepasscx
url: https://devopstales.github.io/linux/docker-credential-in-keepassxc/
date: 2023-03-10
keywords: keepassxc, docker credentials
---


In this post I will show you how to use KeePassXC to store your docker credentials on Linux.

<!--more-->

### Disable gnome-keyring

Ubuntu use gnome-keyring for secret store so first we need to disable this:

```bash
nano /etc/pam.d/gdm-password
# session optional        pam_gnome_keyring.so auto_start
```

```bash
mkdir -p ~/.config/autostart
cp /etc/xdg/autostart/gnome-keyring-*.desktop ~/.config/autostart/

echo 'X-GNOME-Autostart-enabled=false' >> ~/.config/autostart/gnome-keyring-*.desktop
```

### Install docker-credential-pass

```bash
echo SSH_AUTH_SOCK DEFAULT="${XDG_RUNTIME_DIR}/ssh-agent.socket" >> ~/.pam_environment

wget https://github.com/docker/docker-credential-helpers/releases/download/v0.7.0/docker-credential-pass-v0.7.0.linux-amd64

mv docker-credential-pass-v0.7.0.linux-amd64 docker-credential-pass
chmod u+x docker-credential-pass
sudo mv docker-credential-pass /usr/local/bin/docker-credential-pass
```

```bash
cat <<EOF > $HOME/.docker/config.json
{
  "credsStore": "secretservice"
}
EOF
```

### Configure KeePassXC

* First, check the **Enable KeePassXC Freedesktop.org Secret Service integration** box in **Tools > Settings > Secret Service Integration**. This enables the integration at the application level.
* Then, open your password database, go into **Database > Database Settings > Secret Service Integration** and set up a folder to expose over the Secret Service API. You’ll probably want to use a new, empty folder for that.
* We disabled the `gnome-keyring` service because it is interface with the Secret Service API.

### Integration with Docker

Docker team supplies a credential helper that implements the Secret Service protocol already. So we need to install the `docker-credential-secretservice` on the path

```bash
wget https://github.com/docker/docker-credential-helpers/releases/download/v0.7.0/docker-credential-pass-v0.7.0.linux-amd64
mv docker-credential-pass-v0.7.0.linux-amd64 docker-credential-pass
chmod +x docker-credential-pass
mv docker-credential-pass /usr/local/bin/

wget https://github.com/docker/docker-credential-helpers/releases/download/v0.7.0/docker-credential-secretservice-v0.7.0.linux-amd64
mv docker-credential-secretservice-v0.7.0.linux-amd64 docker-credential-secretservice
chmod +x docker-credential-secretservice
mv docker-credential-secretservice /usr/local/bin/
```

Configure docker to use the secret service:

```bash
mkdir $HOME/.docker
echo "{
  "credsStore": "secretservice"
}" > $HOME/.docker/config.json
```

Test with dummy credentials:

```bash
secret-tool store --label='Test test' account cred-test
```

