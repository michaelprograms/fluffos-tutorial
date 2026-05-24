# FluffOS MUD Server — Cheatsheet

Quick-reference for post-setup maintenance. See [README.md](README.md) for full setup instructions.

## MUD Service

```sh
systemctl status mud
systemctl start mud
systemctl stop mud
systemctl restart mud
journalctl -e -u mud                    # view recent logs
journalctl -f -u mud                    # follow live logs
telnet localhost <Telnet Port>
telnet-ssl -z ssl localhost <TLS Telnet Port>
```

## Nginx

```sh
nginx -t                                # test config
systemctl reload nginx                  # reload config (no downtime)
systemctl restart nginx
systemctl status nginx
```

## Certbot

```sh
certbot renew --dry-run                 # test renewal without applying
certbot renew                           # uses plugin configured at issuance
```

## fail2ban

```sh
fail2ban-client status sshd             # check SSH jail
fail2ban-client set sshd unbanip <IP>   # unban an IP
systemctl status fail2ban
systemctl restart fail2ban              # apply jail.local changes
```

## System Maintenance

```sh
df -h                                   # disk usage by partition
du -sh /var/log/* 2>/dev/null           # log directory sizes
journalctl --disk-usage                 # journald disk usage
free -h                                 # memory usage
uptime                                  # load average
apt-get update && apt-get upgrade       # update all packages
apt-get autoremove && apt-get autoclean
```
