---
title: Use acme.sh to issue and install Lets Encrypt certs for multiple subdomains
date: 2020-05-31 14:30:00 Z
categories:
- ssl
tags:
- nginx
- letsencrypt
- acme.sh
---

I like to use the certificate generation tool from [acme.sh](https://acme.sh) as it is just a shell script and uses basic linux utility programs to get everything setup instead of python and all it's dependencies certbot uses. I use nginx for my webserver but prefer to edit the config files myself instead of letting the certificate tool make changes to them automatically so I use the standalone method also.

Install as root (note: don't run as  a single command, $HOME will not change to /root under normal sudo configuration for debian). This will also install a cronjob for root renewing the certs in the future for you automatically.
```shell
sudo -s
apt install socat # acme.sh requires this to launch a standalone web server
curl https://get.acme.sh | sh
source .bashrc # acme.sh puts its scripts on the PATH in .bashrc but this won't change until you relogin or source it like this
```

Issue certs with a standalone webserver. Make sure you stop apache/nginx/etc first as it requires port 80 to be available.
```shell
acme.sh --issue --standalone -d sub1.xoce.kim -d sub2.xoce.kim -d sub3.xoce.kim
```

Install certificates for all subdomains into /etc/ssl/private and set a command to reload the webserver when the cronjobs update the certificate in the future
```shell
acme.sh --install-cert --cert-file /etc/ssl/private/xoce.kim-privkey.pem --fullchain-file /etc/ssl/private/xoce.kim-fullchain.pem -d sub1.xoce.kim -d sub2.xoce.kim -d sub3.xoce.kim --reloadcmd "systemctl reload nginx"
```

Now to setup the webserver configs to use ssl, here is a simple nginx config with http to https redirection
```text
server {
	listen 80;
	server_name sub3.xoce.kim;
	return 301 https://$server_name$request_uri;
}

server {
	server_name sub3.xoce.kim;
	listen 443 ssl;
	ssl_certificate /etc/ssl/private/xoce.kim-fullchain.pem;
	ssl_certificate_key /etc/ssl/private/xoce.kim-privkey.pem;

	location / {
		root /var/www/html;
	}
}
```

Restart the webserver and we are done
```shell
systemctl restart nginx
```
