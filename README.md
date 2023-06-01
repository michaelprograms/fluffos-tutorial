# Fluffos Tutorial

The goal of this tutorial is to document the setup of a server running:
* the fluffos driver
* a basic mudlib
* a website

This tutorial was inspired by the following LDMud Tutorial: https://github.com/cpu/ldmud-tutorial

As such, this tutorial is opinionated and is based on Ubuntu, Apache2, Certbox, and systemd.

# Overview

0. [Prerequisites](#prerequisites)
1. [Server Creation](#server-creation)
2. [DNS Records](#dns-records)
3. [Server Setup](#server-setup)
4. [Website Setup](#website-setup)
5. [Git Setup](#git-setup)
6. [Driver and Mudlib](#driver-and-mudlib-setup)
7. [HTTPS Setup](#https-setup)
8. [Systemd Service](#systemd-service)
9. [Test Connections](#test-connections)
10. [Future Updates](#future-updates)

# Prerequisites

This tutorial assumes you have the following:
* a Digital Ocean account
* a domain name
* command line experience

# Server Creation

Create the (second) smallest available Digital Ocean droplet (~$6 USD a month).

Configuration options:
- Ubuntu 22.10 (LTS) x64
- 1 vCPU, 1 GB RAM, 25 GB disk

After creation:
1. Add your SHH key.
2. Make note of `Server IPv4 address`.

*Note: Fluffos needs 1 GB RAM to build.*

# DNS Records

Add an "A" record with `Server IPv4 address` for `Server Domain Name`.
```
@ A ###.###.###.###
```

# Server Setup

Connect as root user:
```sh
ssh root@`Server Domain Name`
```

Update existing software:
```sh
apt-get update -yy && apt-get upgrade -yy && apt-get dist-upgrade -yy
```

Install required software:
```sh
apt-get install -y build-essential autoconf automake bison \
  cmake libpq-dev libtool libz-dev libgtest-dev libicu-dev \
  libjemalloc-dev libmysqlclient-dev libsqlite3-dev \
  libpcre3-dev libssl-dev telnet telnet-ssl apache2 git
```

Setup non-root user:
```sh
adduser mud

mkdir ~mud/.ssh
cp ~/.ssh/authorized_keys ~mud/.ssh/
chown -R mud:mud ~mud/.ssh
```

Reboot server after updates:
```sh
systemctl reboot
```

# Website Setup

Connect as root user:
```sh
ssh root@`Server Domain Name`
```

Setup website root directory:
```sh
mkdir /var/www/mud

cat >> /var/www/mud/index.html << EOF
<html>
  <head><title>MUD Website</title>
  </head>
  <body>MUD Website</body>
</html>
EOF

chown -R mud:www-data /var/www/mud
```

Setup Apache:
```sh
vi /etc/apache2/sites-available/mud.conf
```

```
<VirtualHost *:80>
  ServerName    `Server Domain Name`
  DocumentRoot  /var/www/mud/
</VirtualHost>
```

Disable default, enable mud, restart.
```sh
a2dissite 000-default

a2ensite mud

systemctl reload apache2
```

http://`Server Domain Name` should connect and display.

# Git Setup

Connect as mud user:
```sh
ssh mud@`Server Domain Name`
```

Setup git:
```sh
git config --global user.email "your@email"
git config --global user.name "Your Name"
```

Setup github key:
```sh
ssh-keygen -t ed25519 -C "your@email"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```
Also copy ~/.ssh/id_ed25519.pub to your GitHub SSH keys: https://github.com/settings/keys

# Driver and Mudlib Setup

Connect as mud user:
```sh
ssh mud@`Server Domain Name`
```

Download mudlib with driver submodule:
```sh
git clone --recurse-submodules git@github.com:fluffos/nightmare3.git
```
*Note: Fluffos driver version included with nightmare3 may be outdated.*

Setup for mudlib shortcut/permissions:
```sh
ln -s /home/mud/nightmare3/ /home/mud/game
chown -R mud:mud ~mud/game
find . -type d -exec chmod -R g+s {} \;
```

Build the fluffos driver:
```sh
cd game
./build.sh
```

Adjust mudlib config:
```sh
vi /home/mud/game/nm3.cfg
```

```
# absolute pathname of mudlib
mudlib directory : /home/mud/game/lib
```

Adjust

# HTTPS Setup

Connect as root user:
```sh
ssh root@`Server Domain Name`
```

Setup Certbox:
```sh
snap install core
snap refresh core
snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot
```

Utilize the LDMud deploy hook to automatically copy renewed certs into mudlib directory:
```sh
mkdir -p /etc/letsencrypt/renewal-hooks/deploy/

curl -o /etc/letsencrypt/renewal-hooks/deploy/fluffos-hook \
  https://gist.githubusercontent.com/cpu/bec1601816db34bb8c9efeb3f78b37c5/raw/c73c7a0b5ce47318710227d25defcf5ae38fc209/ldmud-hook.py

chmod +x /etc/letsencrypt/renewal-hooks/deploy/fluffos-hook

certbot --apache
```

Seed initial certificates to mudlib:
```
certbot --force-renewal

# @TODO verify this section
cp /etc/letsencrypt/live/`Server Domain Name`/fullchain.pem ~mud/game/cert.pem
cp /etc/letsencrypt/live/`Server Domain Name`/chain.pem ~mud/game/issuer.pem
cp /etc/letsencrypt/live/`Server Domain Name`/privkey.pem ~mud/game/key.pem
chown mud:mud ~mud/game/lib/*.pem
```

https://`Server Domain Name` should connect and display.

# Systemd Service

Connect as root user:
```sh
ssh root@`Server Domain Name`
```

Create `/etc/systemd/system/mud.service`:
```
[Unit]
Description = Fluffos MUD Driver
After = network-online.target

[Service]
Type = simple
User = mud
Group = mud
WorkingDirectory = /home/mud/game
ExecStart = /home/mud/game/run.sh
Restart=always
RestartSec=5
OOMScoreAdjust=-900

[Install]
WantedBy = multi-user.target
```

Reload, manually start, check status:
```sh
systemctl daemon-reload
systemctl start mud
systemctl status mud
```

Fluffos MUD Driver output:
```sh
journalctl -e -u mud
```

Enable MUD service at restart:
```sh
systemctl enable mud
```

# Test Connections

Connect as root user:
```sh
ssh root@`Server Domain Name`
```

Test telnet:
```sh
telnet localhost `Telnet Port`
```

Test SSL telnet:
```sh
telnet-ssl -z ssl localhost `TLS Telnet Port`
```

# Future Updates

This tutorial could be improved with the following updates:
* nightmare3 - verify websocket web client works
* nightmare3 - should display if user is connected via TLS port
* nightmare3 - update driver version for efun::sys_refresh_tls(TLS_PORT)
* ldmud-hook.py - possibility of fork for fluffos
