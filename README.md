# FluffOS Tutorial

The goal of this tutorial is to document the setup:
* a droplet running Linux
* using the [FluffOS driver](https://github.com/fluffos/fluffos)
* running the ([nightmare3 mudlib](https://github.com/fluffos/nightmare3))
* and a website
* with TLS

This tutorial was inspired by the following LDMud Tutorial: https://github.com/cpu/ldmud-tutorial

As such, this tutorial is opinionated and is based on Debian/Ubuntu, Apache2, Certbox, and systemd.

# Overview

0. [Prerequisites](#prerequisites)
1. [Server Creation](#server-creation)
2. [DNS Records](#dns-records)
3. [Server Setup](#server-setup)
4. [Website Setup](#website-setup)
5. [Git Setup](#git-setup)
6. [Driver and Mudlib Setup](#driver-and-mudlib-setup)
7. [Apache TLS Setup](#apache-tls-setup)
8. [FluffOS TLS Setup](#fluffos-tls-setup)
9. [Systemd Service Setup](#systemd-service-setup)
10. [Test Connections](#test-connections)
11. [Future Updates](#future-updates)

# Prerequisites

This tutorial assumes you have the following:
* a Digital Ocean account
* a domain name to use for `Server Domain Name`
* Linux and command line experience

# Server Creation

Create the smallest Droplet that satisfies the below RAM requirements. Debian will require less RAM than Ubuntu.

__Debian:__ 1 GB RAM (~$6 USD per month)

__Ubuntu:__ 2 GB RAM (~$12 USD per month)

*Note: Ubuntu will require more RAM than Debian.*

After creation:
1. [Add your SSH key](https://docs.digitalocean.com/products/droplets/how-to/add-ssh-keys/).
2. Make note of `Server IPv4 address`.

# DNS Records

Add an "A" record with `Server IPv4 address` for your `Server Domain Name`.
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

__Debian:__
```sh
apt-get install -y build-essential autoconf automake bison cmake git telnet \
  telnet-ssl libpq-dev libtool libz-dev libgtest-dev libicu-dev libjemalloc-dev \
  libsqlite3-dev libpcre3-dev libssl-dev apache2 default-libmysqlclient-dev
```

__Ubuntu:__
```sh
apt-get install -y build-essential autoconf automake bison cmake git telnet \
  telnet-ssl libpq-dev libtool libz-dev libgtest-dev libicu-dev libjemalloc-dev \
  libsqlite3-dev libpcre3-dev libssl-dev apache2 libmysqlclient-dev
```

Setup non-root user:
```sh
adduser mud
```

Copy SSH key from root user:
```sh
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
```
```sh
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```
Copy `~/.ssh/id_ed25519.pub` to your GitHub SSH keys: https://github.com/settings/keys

# Driver and Mudlib Setup

Connect as mud user:
```sh
ssh mud@`Server Domain Name`
```

Download mudlib with driver submodule:
```sh
git clone --recurse-submodules git@github.com:fluffos/nightmare3.git
```
*Note: FluffOS driver version included with nightmare3 may be outdated.*

Setup for mudlib shortcut/permissions:
```sh
ln -s /home/mud/nightmare3/ /home/mud/game
chown -R mud:mud ~mud/game
find . -type d -exec chmod g+s {} \;
```

Build the FluffOS driver:
```sh
cd game
./build.sh
```

Adjust mudlib config:
```sh
vi /home/mud/game/nm3.cfg
```

Update mudlib directory to the correct absolute path:
```
# absolute pathname of mudlib
mudlib directory : /home/mud/game/lib
```

# Apache TLS Setup

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

https://`Server Domain Name` should connect and display.

# FluffOS TLS Setup

Seed initial certificates to mudlib:
```
certbot --force-renewal
```

If it doesn't work, you can manually set up the initial files:
```
cp /etc/letsencrypt/live/`Server Domain Name`/fullchain.pem ~mud/game/cert.pem
cp /etc/letsencrypt/live/`Server Domain Name`/chain.pem ~mud/game/issuer.pem
cp /etc/letsencrypt/live/`Server Domain Name`/privkey.pem ~mud/game/key.pem
chown mud:mud ~mud/game/lib/*.pem
```

Adjust mudlib config:
```sh
vi /home/mud/game/nm3.cfg
```

Add a telnet port with TLS, pointing to the certificates from the previous step:
```
external_port_2: telnet 6667
external_port_2_tls: cert=cert.pem key=key.pem
```

# Systemd Service Setup

Connect as root user:
```sh
ssh root@`Server Domain Name`
```

Create `/etc/systemd/system/mud.service`:
```
[Unit]
Description = FluffOS MUD Driver
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

To check FluffOS MUD Driver output:
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
* nightmare3 - update driver version for efun::sys_refresh_tls(external_port_# of TLS_PORT)
* ldmud-hook.py - possibility of fork for FluffOS
* automated script for initial server setup
