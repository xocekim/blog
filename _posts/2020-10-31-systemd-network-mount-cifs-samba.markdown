---
title: Mount Windows/Samba/CIFS via SystemD unit
date: 2020-10-31 03:00:00 Z
categories:
- linux
tags:
- mount
- network
---

Create a `/etc/samba/credentials` file with this format
```text
username=myusername
password=mypassword
domain=mydomain // optional
```
Set the permissions for it to read only for root.
`chmod 400 /etc/samba/credentials`

Create a `/etc/systemd/system/mnt-myshare.mount`. Setting `vers=3.1` seems to be the latest supported by CIFS. `uid=myuser,gid=myuser` sets the read write permissions - in this case only `myuser` will be able to access to share.

```text
[Unit]
Description=myshare
Requires=network-online.target
After=network-online.service

[Mount]
What=//192.168.1.100/myshare
Where=/mnt/myshare
Options=credentials=/etc/samba/credentials,vers=3.1,uid=myuser,gid=myuser
Type=cifs

[Install]
WantedBy=multi-user.target
```

Enable with `systemctl enable mnt-myshare.mount` and start `systemctl start mnt-myshare.mount`.
See any errors with `journalctl -u mnt-myshare.mount`

There can be a requirement to install smbutils for credentials file reading to work.
