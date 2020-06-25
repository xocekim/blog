---
title: My Wireguard setup with IPv4/IPv6 dual stack using a VPS, debian, liquorix
  kernel, nftables and DoH
date: 2020-06-25 14:30:00 Z
categories:
- vpn
- firewall
- networking
- dns
tags:
- wireguard
- systemd
- nftables
- debian
- dns-over-https
- doh
- ipv6
---

Instead of using a VPN provider I like to host my own on a cheap VPS. I like to use (Debian)[https://www.debian.org] as my distro as it's highly documented and kept minimal by default and with systemd to configure most things. I install the [liquorix](https://liquorix.net) kernel because it is a bit newer than the debian stable kernel and has support for Wireguard built in since 5.6 (debian stable is still on 4.19 at the time of writing). I will use the (NextDNS)[https://github.com/nextdns/nextdns] DNS-Over-HTTPS client to send my DNS queries securely to (Adguard)[https://adguard.com]. Finally, I am securing all this with (nftables)[https://www.netfilter.org/projects/nftables/] for the firewall as this seems to be superseding iptables now and again is built right into the kernel.

First lets gain root make sure everything is up to date and install the liquorix kernel
```shell
sudo -s
apt update
apt dist-upgrade
echo "deb http://liquorix.net/debian buster main" | tee /etc/apt/sources.list.d/liquorix.list
curl 'https://liquorix.net/linux-liquorix.pub' | apt-key add - && apt-get update
apt install linux-image-liquorix-amd64 linux-image-liquorix-headers-amd64 # you don't need headers for this tutorial but your probably going to want them at some point if your building anything from source
systemctl reboot # reboot into new kernel
uname -r # should now show your running the liquorix kernel
```

Next up is networking, we are committing to systemd-networkd for this so lets move everything over.
Lets just look at our current configuration to see what our IP addresses are and what interfaces they are on and install wireguard-tools to see some Wireguard info and be able to generate public/private keypairs later
```shell
ip a
```
Assuming your /etc/network/interfaces looks like this
```text
auto lo
iface lo inet loopback
auto eth0
iface eth0 inet static
	address 192.0.2.7/24
	gateway 192.0.2.254
iface eth0 inet6 static
	address 2001:db8::c0ca:1eaf/64
	gateway 2001:db8::1ead:ed:beef
```

We are going to create a new file `/etc/systemd/network/eth0.network` and fill it out like this
```text
[Match]
Name=eth0

[Network]
Address=192.0.2.7/24
Gateway=192.0.2.254

Address=2001:db8::c0ca:1eaf/64
Gateway=2001:db8::1ead:ed:beef
```

We don't need to worry about the loopback interface as systemd sorts that for us. Lets disable the old setup and enable the new one.
# WARNING if your doing this remotely and this setup has a problem you are about to lock yourself out of your server so make sure you have a way back in such as logging on to your VPS providers control panel and using a VNC terminal
```shell
systemctl disable networking.service
systemctl enable systemd-networkd.service
```
I like to reboot at this point and double check the networking is all good by pinging cloudflare public servers
```shell
systemctl reboot
ip a # should look same as the last time we ran this
ping 1.1.1.1 # success: IPv4 working
ping 2606:4700:4700::1111 # success: IPv6 working
ping -4 one.one.one.one # success: IPv4 DNS is resolving
ping -6 one.one.one.one # success: IPv6 DNS is resolving
```

Assuming all is well lets now create the Wireguard interface. We need 2 files for this - a network file for the interface and a configuration file holding the interfaces configuration. First is the interface itself at `/etc/systemd/network/wg0.network`
```text
[Match]
Name=wg0

[Network]
Address=10.46.46.1/24
Address=fd46:46:46::1/64
```
And for the configuration were going to need some public/private key pairs.
```shell
wg genkey | tee server.key | wg pubkey > server.pub
wg genkey | tee client1.key | wg pubkey > client1.pub
wg genkey | tee client2.key | wg pubkey > client2.pub
```
In the configuration you should replace X_KEY with the contents of X.key and X_PUBKEY with the contents of X.pub
Lets create the configuration now at `/etc/systemd/network/wg0.netdev`
```text
[NetDev]
Name=wg0
Kind=wireguard

[WireGuard]
PrivateKey=SERVER_KEY
ListenPort=51820

[WireGuardPeer]
PublicKey=CLIENT1_PUBKEY
AllowedIPs=10.46.46.2/32
AllowedIPs=fd46:46:46::2/128

[WireGuardPeer]
PublicKey=CLIENT2_PUBKEY
AllowedIPs=10.46.46.3/32
AllowedIPs=fd46:46:46::3/128
```

Hop on over to (NextDNS on Github)[https://github.com/nextdns/nextdns/releases] and grab the latest release. I am assuming you are on x86_64 architecture and the latest release at the time of writing is v1.7.0
```shell
wget "https://github.com/nextdns/nextdns/releases/download/v1.7.0/nextdns_1.7.0_linux_amd64.deb"
dpkg -i nextdns_1.7.0_linux_amd64.deb
```
Now I am going to set nextdns to only listen on our wg0 interface, setup a cache of 8MB and use (Adguard)[https://adguard.com] as my resolver then enable and start the service
```shell
nextdns config set -listen 10.46.46.1:53 -cache-size 8MB -forwarder "https://dns.adguard.com/dns-query"
systemctl enable --now nextdns.service
```

Finally the firewall configuration in `/etc/nftables.conf`. This is a basic server firewall that allows TCP port 22 for using SSH and UDP ports 53 and 51820 for our DNS server and Wireguard VPN respectively. We respond to pings and we play nice with IPv6 neighbour discovery. We forward packets from wg0 (our VPN network) out through eth0 (the WAN). I have had to duplicate it for IPv6 as it would appear there is no support currently for masquerading both IPv4 and IPv6 in a single table using the inet family rather than ip/ip6. Hopefully I find a way around this or this is implemented soon as it makes the config much larger than it needs to be!
```text
#!/usr/sbin/nft -f

flush ruleset

table ip firewall {
        chain incoming {
                type filter hook input priority 0; policy drop;
                iif { lo, wg0 } accept
                ct state established,related accept
                ct state invalid drop
        ip protocol icmp icmp type { destination-unreachable, echo-reply, echo-request, source-quench, time-exceeded } accept
                tcp dport { 22 } accept
                udp dport { 53, 51820 } accept
        }

        chain forward {
                type filter hook forward priority 0; policy drop;
				iif eth0 oif wg0 ct state { related, established } accept
				iif wg0 oif eth0 accept
        }

        chain postrouting {
                type nat hook postrouting priority 100; policy accept;
				ip saddr 10.46.46.0/24 oif eth0 masquerade
        }

}

table ip6 firewall {
        chain incoming {
                type filter hook input priority 0; policy drop;
                iif { lo, wg0 } accept
                ct state established,related accept
                ct state invalid drop
				ip6 nexthdr icmpv6 icmpv6 type { destination-unreachable, echo-reply, echo-request, nd-neighbor-solicit, nd-router-advert, nd-neighbor-advert, packet-too-big, parameter-problem, time-exceeded } accept
                tcp dport { 22 } accept
                udp dport { 53, 51820 } accept
        }

        chain forward {
                type filter hook forward priority 0; policy drop;
				iif eth0 oif wg0 ct state { related, established } accept
				iif wg0 oif eth0 accept
        }

        chain postrouting {
                type nat hook postrouting priority 100; policy accept;
				ip6 saddr fd46:46:46::0/64 oif eth0 masquerade
        }
}
```
Test the config before applying it, we don't want to lock ourselved out of our server! Then lets turn it all on.
```shell
nft -cf /etc/nftables
systemctl enable --now nftables.service
```


