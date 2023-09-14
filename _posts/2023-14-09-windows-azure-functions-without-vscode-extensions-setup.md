---
title: Setup azure python serverless functions environment without using vscode extensions
date: 2023-09-14
categories:
- windows
tags:
- python
- azure
- functions
---

Install required CLI packages using winget
```powershell
winget install Microsoft.AzureCLI Microsoft.Azure.FunctionsCoreTools OpenJS.NodeJS.LTS
npm install -g azurite
```

Configure azure account
```powershell
az login
az storage account create --name MYSTORAGEACCOUNT --resource-group MYRESOURCEGROUP --sku Standard_LRS
az functionapp create --os-type Linux --runtime python --name MYFUNCNAME --resource-group MYRESOURCEGROUP --storage-account MYSTORAGEACCOUNT --consumption-plan-location MYREGION
```

We can set environment variables for the function like this
```powershell
az functionapp config appsettings set --name MYFUNCNAME -g MYRESOURCEGROUP --settings "MY_SECRET=password"
```

If we are using cosmosdb
```powershell
az cosmosdb create --resource-group MYRESOURCEGROUP --name MYCOSMOSDBNAME
az cosmosdb keys list --resource-group MYRESOURCEGROUP --name MYCOSMOSDBNAME --type "connection-strings"
az functionapp config appsettings set --name MYFUNCNAME -g MYRESOURCEGROUP --settings "COSMOS_CONNECTION_STRING=MYCOSMOS_CONNECTION_STRING"
```

Create local function app
```powershell
func new MYFUNCNAME -n MYFUNCNAME -t "HTTP trigger" -m V2 --no-docs
cd MYFUNCNAME
python3 -mvenv venv
.\venv\Scripts\Activate.ps1
python -mpip install --upgrade pip azure-functions
```

Start azurite for storage
```powershell
cd MYFUNCNAME
azurite --location azurite
```

Start the func
```powershell
func start
```