---
layout: post
title:  "Setting up a factorio server on Azure Container Instances"
---

Guide to setup server for [Factorio](https://factorio.com/) game.
Use [Azure Cload Shell](https://portal.azure.com/#cloudshell/) to run commands.

### [Azure](https://azure.microsoft.com/en-us/)
1. Resource group
``` bash
rg=rg-factorio-server
location=westeurope
az group create -l $location -n $rg
```
2. Azure files share
``` bash
storage=stfactorio$RANDOM
share=sharefactorio
az storage account create -g $rg -n $storage -l $location --sku Standard_LRS
az storage share create -n $share --account-name $storage
```
3. Save ARM template deploy-factrio-server.json
Get your storage key executing command
``` bash
STORAGE_KEY=$(az storage account keys list --resource-group $rg --account-name $storage --query "[0].value" --output tsv)
```
Replace values before running
``` json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "variables": {
    "containername": "container-factorio-server",
    "containerimage": "docker.io/factoriotools/factorio:stable"
  },
  "resources": [
    {
      "name": "ci-factorio-server-<unique name>",
      "type": "Microsoft.ContainerInstance/containerGroups",
      "apiVersion": "2021-03-01",
      "location": "[resourceGroup().location]",
      "properties": {
        "sku": "Standard",
        "containers": [
          {
            "name": "[variables('containername')]",
            "properties": {
              "image": "[variables('containerimage')]",
              "resources": {
                "requests": {
                  "cpu": 1,
                  "memoryInGb": 1.5
                }
              },
              "ports": [
                {
                  "protocol": "TCP",
                  "port": 27015
                },
                {
                  "protocol": "UDP",
                  "port": 34197
                }
              ], 
              "volumeMounts": [
                {
                  "name": "filesharevolume",
                  "mountPath": "/factorio"
                }
              ],
              "environmentVariables": [
                {
                  "name": "LOAD_LATEST_SAVE",
                  "value": "false"
                },
                {
                  "name": "GENERATE_NEW_SAVE",
                  "value": "true"
                },
                {
                  "name": "SAVE_NAME",
                  "value": "generatenewsave"
                }
              ]
            }
          }
        ],
        "osType": "Linux",
        "restartPolicy": "Always",
        "ipAddress": {
          "type": "Public",
          "ports": [
            {
              "protocol": "tcp",
              "port": 27015
            },
            {
              "protocol": "udp",
              "port": 34197
            }
          ],
          "dnsNameLabel": "factorio-server-<unique name>"
        },
        "volumes": [
          {
            "name": "filesharevolume",
            "azureFile": {
                "shareName": "sharefactorio",
                "storageAccountName": "<storage acccount name>",
                "storageAccountKey": "<storage account key>"
            }
          }
        ]
      }
    }
  ]
}
```
4. Run deployment
``` bash
az deployment group create --resource-group $rg --template-file deploy-factorio-server.json
```

5. Exec bash inside container and attach to it
``` bash
az container exec -g $rg -n <container instances name> --container-name <container name> --exec-command "sh"
```

6. Edit factorio/config/server-settings.json (mandatory to set username + password/token)

### Links
1. https://github.com/factoriotools/factorio-docker
2. https://docs.microsoft.com/en-us/azure/storage/files/storage-how-to-use-files-cli
3. https://docs.microsoft.com/en-us/azure/container-instances/container-instances-volume-azure-files
