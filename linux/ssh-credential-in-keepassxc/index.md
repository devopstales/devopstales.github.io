---
title: Store your ssh keys in keepassxc
url: https://devopstales.github.io/linux/ssh-credential-in-keepassxc/
date: 2023-03-15
keywords: keepassxc, ssh keys, ssh agent, security
---


In this post I will show you how to use KeePassXC to store your ssh credentials.

<!--more-->

### Configure ssh agent:

```bash
# add to ~/.zshrc or ~/.bashrc
cat <<EOF >> ~/.zshrc
if ! pgrep -u "$USER" ssh-agent > /dev/null; 
then    
    ssh-agent > "$XDG_RUNTIME_DIR/ssh-agent.env" 
fi 
if [[ ! "$SSH_AUTH_SOCK" ]]; 
then     
    eval "$(<"$XDG_RUNTIME_DIR/ssh-agent.env")" 
fi
EOF
```

```bash
mkdir -p ~/.config/systemd/user/

cat <<EOF > ~/.config/systemd/user/ssh-agent.service
[Unit]
Description=SSH key agent

[Service]
Type=simple
Environment=SSH_AUTH_SOCK=%t/ssh-agent.socket
# DISPLAY required for ssh-askpass to work
Environment=DISPLAY=:0
ExecStart=/usr/bin/ssh-agent -D -a $SSH_AUTH_SOCK

[Install]
WantedBy=default.target
EOF
```

```bash
echo SSH_AUTH_SOCK DEFAULT="${XDG_RUNTIME_DIR}/ssh-agent.socket" >> ~/.pam_environment
```

Remember that you’ll need to re login for the `.pam_environment` changes to take effect.

### Configuring KeepassXC for SSH Agent Integration

* Ppen KeepassXC and go to **Tools > Settings**. In the settings window click on **SSH Agent**, and then tick the checkbox that says **Enable SSH Agent**.

![KeepassXC-Add-entry](/img/include/KeepassXC-SSH-agent.webp)

Now we need to add the ssh key to keepassxc:

![KeepassXC-Add-entry](/img/include/KeepassXC-Add-entry.webp)

If you have a password protected key add the password to keepassxc without a username.

![KeepassXC-Add-key](/img/include/KeepassXC-Add-key.webp)

And the last thing you need to do is to set up the key to be available to the SSH Agent. For that just click the SSH Agent icon:

![KeepassXC-Add-SSH-Agent-entry](/img/include/KeepassXC-Add-SSH-Agent-entry.webp)

### Starting SSH Agent with systemd

```bash
echo '[Unit]
Description=SSH key agent

[Service]
Type=simple
Environment=SSH_AUTH_SOCK=%t/ssh-agent.socket
# DISPLAY required for ssh-askpass to work
Environment=DISPLAY=:0
ExecStart=/usr/bin/ssh-agent -D -a $SSH_AUTH_SOCK

[Install]
WantedBy=default.target
' >  ~/.config/systemd/user/ssh-agent.service 
```

Now you can start and stop the SSH Agent using:

```bash
systemctl start --user ssh-agent
systemctl enable --user ssh-agent
```

Lock and unlock your database to see that the key is now available in your agent:

```bash
ssh-add -l
256 SHA256:pPcl+pI2TTkzIL6psB/13wwxQLOA9jZp1+A/E+sHI3Q bastion production (ED25519)
```

For OSX you need to start a separete ssh-agent process:

```bash
eval $(ssh-agent)
open -a KeePassXC.app /dev/null --args --allow-screencapture
```

