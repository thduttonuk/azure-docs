---
title: Azure Resource Manager template samples for chaos experiments
description: Sample Azure Resource Manager templates to create Azure Chaos Studio experiments
services: chaos-studio
author: johnkemnetz
ms.topic: sample
ms.date: 11/10/2021
ms.author: johnkem
ms.service: chaos-studio
---

# Resource Manager template samples for experiments in Azure Chaos Studio
This article includes sample [Azure Resource Manager templates](../azure-resource-manager/templates/syntax.md) to create a [chaos experiment](chaos-studio-chaos-experiments.md) in Azure Chaos Studio. Each sample includes a template file and a parameters file with sample values to provide to the template.

## Create experiment (sample)

In this sample, we create a chaos experiment with a single target resource and a single CPU pressure fault. The experiment can be modified by referencing our [REST API](/rest/api/chaosstudio/experiments/create-or-update) and [fault library](chaos-studio-fault-library.md).

### Template file

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "experimentName": {
      "type": "string",
      "defaultValue": "simple-experiment",
      "metadata": {
        "description": "A name for the experiment."
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]",
      "metadata": {
        "description": "A region where the experiment will be deployed."
      }
    },
    "chaosTargetResourceId": {
      "type": "string",
      "metadata": {
        "description": "Resource ID of the chaos target. This is the resource ID of the resource being targeted plus the target proxy resource."
      }
    }
  },
  "functions": [],
  "variables": {},
  "resources": [
    {
      "type": "Microsoft.Chaos/experiments",
      "apiVersion": "2021-09-15-preview",
      "name": "parameters('experimentName')",
      "location": "parameters('location')",
      "identity": {
        "type": "SystemAssigned"
      },
      "properties": {
        "identity": {
          "properties": {
            "type": "SystemAssigned"
          }
        },
        "selectors": [
          {
            "id": "Selector1",
            "type": "List",
            "targets": [
              {
                "type": "ChaosTarget",
                "id": "parameters('chaosTargetResourceId')"
              }
            ]
          }
        ],
        "startOnCreation": "false",
        "steps": [
          {
            "name": "Step1",
            "branches": [
              {
                "name": "Branch1",
                "actions": [
                  {
                    "duration": "PT10M",
                    "name": "urn:csci:microsoft:agent:cpuPressure/1.0",
                    "parameters": [
                      {
                        "key": "pressureLevel",
                        "value": "95"
                      }
                    ],
                    "selectorId": "Selector1",
                    "type": "continuous"
                  }
                ]
              }
            ]
          }
        ]
      }
    }
  ],
  "outputs": {}
}
```

### Parameter file

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
      "experimentName": {
        "value": "my-first-experiment"
      },
      "location": {
        "value": "eastus"
      },
      "chaosTargetResourceId": {
        "value": "/subscriptions/<subscription-id>/resourceGroups/<rg-name>/providers/Microsoft.DocumentDB/databaseAccounts/<account-name>"
      }
  }
}
```

## Next steps

* [Learn more about Chaos Studio](chaos-studio-overview.md).
* [Learn more about chaos experiments](chaos-studio-chaos-experiments.md).
