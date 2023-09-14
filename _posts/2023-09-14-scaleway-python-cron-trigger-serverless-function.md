---
title: Create serverless python function on Scaleway triggered by cron
date: 2023-09-14
categories:
- python
tags:
- windows
- serverless
- scaleway
- functions
---

Install NodeJS

```powershell
winget install OpenJS.NodeJS.LTS
```

Install serverless framework
```powershell
npm install serverless -g
```

Get latest scaleway-cli exe from [Scaleway Github](https://github.com/scaleway/scaleway-cli/releases/) and copy the exe to somewhere on path as scaleway-cli.exe

Login to [Scaleway Console API Keys](https://console.scaleway.com/iam/api-keys) and create a new user and run the command it gives you such as
```powershell
scaleway-cli.exe init -p myprofile access-key=MYACCESSKEY secret-key=MYSECRETKEY organization-id=MYORGID project-id=MYPROFILEID
```

Create our function
```powershell
scaleway-cli.exe function function create name=MYFUNCTIONNAME namespace-id=MYNAMESPACE runtime=python310
```

