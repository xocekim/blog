---
title: Autostart quadlet for miniflux RSS reader
date: 2025-05-10
categories:
- container
tags:
- miniflux
- postgres
- podman
- quadlet
---

Create secrets file for podman with our password
```shell
echo mysecret > /tmp/postgres_pw.txt
```

Create secrets file for podman with our database uri
```shell
echo "postgres://miniflux:mysecret@db/miniflux?sslmode=disable" > /tmp/postgres_uri.txt
```

Create initial SQL to create miniflux user
```sql
-- /tmp/init.sql
CREATE USER miniflux WITH PASSWORD 'miniflux';
CREATE DATABASE miniflux OWNER miniflux;
GRANT ALL PRIVILEGES ON DATABASE miniflux TO miniflux;
```

Create network file for quadlet
```text
# ~/.config/containers/systemd/mynet.network
[Unit]
Description=mynetwork for podman

[Network]
```

Create quadlet container file for postgresql
```text
# ~/.config/containers/systemd/postgres.container
[Unit]
Description=podman postgres container

[Container]
ContainerName=postgres
Image=docker.io/library/postgres:17-alpine
Network=mynet.network
Environment=POSTGRES_PASSWORD_FILE=/run/secrets/postgres_pw
Volume=postgres:/var/lib/postgresql/data
Volume=/tmp/init.sql:/docker-entrypoint-initdb.d/init-miniflux-user.sql:Z
Secret=postgres_pw
```

Create quadlet container file for miniflux

```text
# ~/.config/containers/systemd/miniflux.container
[Unit]
Description=podman miniflux container
Requires=postgres.service
After=postgres.service

[Container]
ContainerName=miniflux
Image=docker.io/miniflux/miniflux:latest
Network=mynet.network
Environment=DATABASE_URL_FILE=/run/secrets/postgres_uri
Environment=ADMIN_USERNAME=admin
Environment=ADMIN_PASSWORD_FILE=/run/secrets/postgres_pw
Environment=CREATE_ADMIN=1
Environment=RUN_MIGRATIONS=1
Secret=postgres_pw
Secret=postgres_uri
```

Create podman secrets from the files we created
```shell
podman secret create postgres_pw /tmp/postgres_pw.txt
podman secret create postgres_uri /tmp/postgres_uri.txt
```

Start it up
```shell
systemctl --user daemon-reload
systemctl --user enable --now miniflux.service
```