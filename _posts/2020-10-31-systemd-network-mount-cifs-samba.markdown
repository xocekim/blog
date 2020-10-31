---
title: Mount Windows/Samba/CIFS via SystemD unit
date: 2020-10-31 03:00:00 Z
categories:
- linux
tags:
- mount
- network
---

Create the folder we will mount to
```shell
mkdir -p /mnt/myshare
```

Create a `/etc/samba/credentials` file with this format
```text
username=myusername
password=mypassword
domain=mydomain // optional
```
Set the permissions for it to read only for root.
`chmod 400 /etc/samba/credentials`

Create a `/etc/systemd/system/mnt-myshare.mount`. Setting `vers=3.1` seems to be the latest supported by CIFS. `uid=myuser,gid=myuser` sets the read write permissions - in this case only `myuser` will be able to access to share. There is a requirement that the name of the share (in this case `myshare`) be the same as the folder mount point so a `mnt-abc123.mount` file would need to be mounted at `/mnt/abc123`.

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

Enable, start and check log
```shell
systemctl enable mnt-myshare.mount
systemctl start mnt-myshare.mount
journalctl -u mnt-myshare.mount
```

There can be a requirement to install smbutils for credentials file reading to work.
