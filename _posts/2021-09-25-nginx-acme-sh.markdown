---
title: Nginx + acme.sh + no cron + systemd-timer
date: 2021-09-25 03:00:00 Z
categories:
- linux
tags:
- nginx
- ssl
---

```shell
export DOMAIN=example.com
export WEBPATH=/var/www/$DOMAIN
mkdir -p $WEBPATH/{htdocs,logs,ssl}
chown -R root:www-data $WEBPATH
curl https://get.acme.sh | sh -s email=my@example.com --force --install
cat > /etc/nginx/sites-available/$DOMAIN <<EOF
server {
    listen       [::]:80 http2;
    # listen       [::]:443 ssl http2;
    server_name  rss.xoce.kim;
    access_log   /var/www/rss/logs/access.log;
    error_log    /var/www/rss/logs/error.log;
    # ssl_certificate /var/www/rss/ssl/fullchain.pem;
    # ssl_certificate_key /var/www/rss/ssl/privkey.pem;
    # if ($scheme != "https") {
    #     return 301 https://$host$request_uri;
    # }
    location / {
        root /var/www/rss/htdocs;
    }
}
EOF
ln -s /etc/nginx/sites-available/$DOMAIN /etc/nginx/sites-enabled/$DOMAIN
acme.sh --issue -d $DOMAIN -w $WEBPATH/htdocs
acme.sh --install-cert -d $DOMAIN --key-file $WEBPATH/ssl/privkey.pem --fullchain-file $WEBPATH/ssl/fullchain.pem --reloadcmd "systemctl reload nginx"
# now uncomment everything in /etc/nginx/sites-available/$DOMAIN
cat > /etc/systemd/system/acme-renew.service <<EOF
[Unit]
Description=Renew acme certs

[Service]
Type=simple
ExecStart=sh -c "/root/.acme.sh/acme.sh --cron --home /root/.acme.sh"

[Install]
WantedBy=default.target
EOF
cat > /etc/systemd/system/acme-renew-timer.timer <<EOF
[Unit]
Description=Renew acme certs every day
RefuseManualStart=no
RefuseManualStop=no

[Timer]
Persistent=true
OnCalendar=*-*-* 02:20:20
Unit=acme-renew.service

[Install]
WantedBy=timers.target
EOF
systemctl enable --now acme-renew.service acme-renew-timer.timer
```

