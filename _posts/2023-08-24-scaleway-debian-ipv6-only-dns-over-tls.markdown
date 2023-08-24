---
title: Setup Debian 12 on a Scaleway IPv6 only instance to use DNS-over-TLS
date: 2023-08-24
categories:
- linux
tags:
- debian
- dns
- scaleway
- ipv6
---

The main problem here being that scaleway uses cloud-init, which then uses netplan, which then sets up systemd-resolved incorrectly because netplan only supports a few parameters to setup the connection.

# edit the netplan and tell it to not use the DHCP provided DNS servers
```shell
vim /etc/netplan/50-cloud-init.yaml
```
```yaml
 ...
 dhcp4: true
 dhcp4-overrides:
   use-dns: false
 dhcp6-overrides:
   use-dns: false
 ...
```
# now set systemd-resolved to use specific DNS server for all interfaces
## here I am using a public DNS64+NAT64 resolver from nat64.dk - you can find more at https://nat64.net/public-providers
```shell
vim /etc/systemd/resolved.conf
```
```ini
...
[Resolve]
DNS=2a00:1098:2b::1#dot.nat64.dk
DNSOverTLS=yes
...
```
# reboot

