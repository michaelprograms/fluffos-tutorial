# FluffOS Tutorial

The goal of this tutorial is to document the setup:
* a droplet running Linux
* using the [FluffOS driver](https://github.com/fluffos/fluffos)
* running the ([nightmare3 mudlib](https://github.com/fluffos/nightmare3))
* and a website
* with TLS

This tutorial was inspired by the following LDMud Tutorial: https://github.com/cpu/ldmud-tutorial

As such, this tutorial is opinionated and is based on Debian/Ubuntu, Nginx, Certbot, and systemd.

# Overview

0. [Prerequisites](#prerequisites)
1. [Server Creation](#server-creation)
2. [DNS Records](#dns-records)
3. [Server Setup](#server-setup)
4. [Website Setup](#website-setup)
5. [No Website Alternative](#no-website-alternative)
6. [Git Setup](#git-setup)
7. [Driver and Mudlib Setup](#driver-and-mudlib-setup)
8. [Nginx TLS Setup](#nginx-tls-setup)
9. [FluffOS TLS Setup](#fluffos-tls-setup)
10. [Systemd Service Setup](#systemd-service-setup)
11. [Test Connections](#test-connections)
12. [Future Updates](#future-updates)

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

*Optional — Ghostty terminal: after first login, verify `$TERM` is recognized on the remote (`toe | grep ghostty`). If it is not listed, deploy Ghostty's terminfo by running the following from your local machine:*
```sh
infocmp xterm-ghostty | ssh root@`Server Domain Name` 'tic -x -'
```
*This fixes line-wrap truncation and history garbling.*

Update existing software:
```sh
apt-get update -yy && apt-get upgrade -yy && apt-get dist-upgrade -yy
```

Install required software:

__Debian:__
```sh
apt-get install -y build-essential autoconf automake bison cmake git telnet \
  telnet-ssl libpq-dev libtool libz-dev libgtest-dev libicu-dev libjemalloc-dev \
  libsqlite3-dev libpcre3-dev libssl-dev nginx default-libmysqlclient-dev fail2ban
```

__Ubuntu:__
```sh
apt-get install -y build-essential autoconf automake bison cmake git telnet \
  telnet-ssl libpq-dev libtool libz-dev libgtest-dev libicu-dev libjemalloc-dev \
  libsqlite3-dev libpcre3-dev libssl-dev nginx libmysqlclient-dev fail2ban
```

Setup non-root user:
```sh
adduser --disabled-password --gecos "" mud
```

`--disabled-password` *locks the account from the start (SSH key auth and* `su - mud` *from root still work).* `--gecos ""` *skips the Full Name / Room Number / etc. prompts.*

Copy SSH key from root user:
```sh
mkdir ~mud/.ssh
cp ~/.ssh/authorized_keys ~mud/.ssh/
chown -R mud:mud ~mud/.ssh
```

Set your timezone if you don't want the default UTC.
```sh
timedatectl list-timezones
```
```sh
timedatectl set-timezone America/New_York
```

Setup fail2ban to ban IPs with repeated failed SSH login attempts:
```sh
vi /etc/fail2ban/jail.local
```

```
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled = true
backend = systemd
```

```sh
systemctl enable fail2ban && systemctl restart fail2ban
```

Verify the SSH jail is active:
```sh
fail2ban-client status sshd
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

Setup Nginx:
```sh
vi /etc/nginx/sites-available/mud
```

```nginx
server {
    listen 80;
    server_name `Server Domain Name`;
    root /var/www/mud;
    index index.html;
}
```

Disable default, enable mud, reload.
```sh
rm /etc/nginx/sites-enabled/default

ln -s /etc/nginx/sites-available/mud /etc/nginx/sites-enabled/

nginx -t && systemctl reload nginx
```

http://`Server Domain Name` should connect and display.

# No Website Alternative

If you do not need a website, skip [Website Setup](#website-setup) and remove `nginx` from the Server Setup package install. Certbot's standalone mode temporarily binds port 80 only during issuance and renewal — no web server process runs otherwise.

Connect as root user:
```sh
ssh root@`Server Domain Name`
```

Install certbot:
```sh
apt-get install -y certbot
```

Obtain certificate:
```sh
certbot certonly --standalone -d `Server Domain Name`
```

Then skip [Nginx TLS Setup](#nginx-tls-setup) down to the deploy hook and `certs_path` steps, which still apply, then continue at [FluffOS TLS Setup](#fluffos-tls-setup).

# Git Setup

Connect as mud user:
```sh
ssh mud@`Server Domain Name`
```

*Optional — the mud user's prompt will have no colors by default. Add a colored PS1 to `~/.bashrc` to distinguish it from root:*
```sh
echo 'PS1="\[\e]0;\u@\h: \w\a\]${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$"' >> ~/.bashrc
source ~/.bashrc
```

Configure git identity for the mud user:
```sh
git config --global user.email "your@email"
git config --global user.name "Your Name"
```

Setup the mud user's GitHub SSH key:
```sh
ssh-keygen -t ed25519 -C "your@email"
```
```sh
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```
Copy the contents of `~/.ssh/id_ed25519.pub` to your GitHub SSH keys: https://github.com/settings/keys

Verify the SSH key was added to GitHub successfully:
```sh
ssh -T git@github.com
```

# Driver and Mudlib Setup

Connect as mud user:
```sh
ssh mud@`Server Domain Name`
```

Download mudlib with driver submodule:
```sh
git clone --recurse-submodules git@github.com:username/`Mudlib Repository`.git
```
*Note: FluffOS driver version included with your mudlib may be outdated.*

*If you don't have your own mudlib, you can use [nightmare3](https://github.com/fluffos/nightmare3) as a starting point:*
```sh
git clone --recurse-submodules git@github.com:fluffos/nightmare3.git
```

Setup for mudlib shortcut/permissions:
```sh
ln -s /home/mud/`Mudlib Repository`/ /home/mud/game
chown -R mud:mud ~mud/game
find /home/mud/game -type d -exec chmod g+s {} +
```

*The `find` command sets the setgid bit on every subdirectory so files created by the MUD driver inherit the `mud` group.*

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

# Nginx TLS Setup

Connect as root user:
```sh
ssh root@`Server Domain Name`
```

Setup Certbot:
```sh
apt-get install -y certbot python3-certbot-nginx
```

Utilize the LDMud deploy hook to automatically copy renewed certs into mudlib directory:
```sh
mkdir -p /etc/letsencrypt/renewal-hooks/deploy/

curl -o /etc/letsencrypt/renewal-hooks/deploy/fluffos-hook \
  https://gist.githubusercontent.com/cpu/bec1601816db34bb8c9efeb3f78b37c5/raw/c73c7a0b5ce47318710227d25defcf5ae38fc209/ldmud-hook.py

chmod +x /etc/letsencrypt/renewal-hooks/deploy/fluffos-hook
```

Adjust the line for `certs_path` in `/etc/letsencrypt/renewal-hooks/deploy/fluffos-hook` to point to where TLS certificates will be stored in the mudlib. For example:
```
certs_path = f"{mud_home}/lib/secure/etc/tls"
```

Then:
```sh
certbot --nginx
```

https://`Server Domain Name` should connect and display.

# FluffOS TLS Setup

Seed initial certificates to mudlib:
```
certbot --force-renewal
```

If it doesn't work, you can manually set up the initial files:
```
cp /etc/letsencrypt/live/`Server Domain Name`/fullchain.pem ~mud/game/lib/secure/etc/tls/`Server Domain Name`.crt
cp /etc/letsencrypt/live/`Server Domain Name`/chain.pem ~mud/game/lib/secure/etc/tls/`Server Domain Name`.issuer.crt
cp /etc/letsencrypt/live/`Server Domain Name`/privkey.pem ~mud/game/lib/secure/etc/tls/`Server Domain Name`.key
chown mud:mud ~mud/game/lib/secure/etc/tls/*.pem
```

Adjust mudlib config:
```sh
vi /home/mud/game/nm3.cfg
```

Add a telnet port with TLS, pointing to the certificates from the previous step:
```
external_port_2: telnet 6667
external_port_2_tls: cert=secure/etc/tls/`Server Domain Name`.crt key=`Server Domain Name`.key
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

Cap journald disk usage and in-memory buffer to reduce RAM and disk usage:
```sh
vi /etc/systemd/journald.conf
```
Add:
```
SystemMaxUse=200M

RuntimeMaxUse=16M
RuntimeMaxFileSize=4M
```
```sh
systemctl restart systemd-journald
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
* UFW - configure firewall to allow only required ports
* unattended-upgrades - automatic security patch application
