---
layout: post
title:  "Setting up a factorio server on azure container instances"
---

Guide to setup server for [Factorio](https://factorio.com/) game.
Use windows bash to run commands. Important: don't use git bash

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
STORAGE_KEY=$(az storage account keys list --resource-group $rg --account-name $storage --query "[0].value" --output tsv)
```
3. Save ARM template deploy-factrio-server.json (make changes before running)
``` json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "variables": {
    "containername": "container-factorio-server",
    "containerimage": "docker.io/dtandersen/factorio:stable"
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
                "storageAccountName": "stfactorio22392",
                "storageAccountKey": "Eu8+BARJdmiVm+PBiuHh6RV9A9SxgNODyWdgDZ7ivjoAjxs7OigF3oQsxa1GyMLAUX6OfvhO/omanUirs0qblQ=="
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

### Links
1. https://github.com/factoriotools/factorio-docker
2. https://docs.microsoft.com/en-us/azure/storage/files/storage-how-to-use-files-cli
3. https://docs.microsoft.com/en-us/azure/container-instances/container-instances-volume-azure-files
