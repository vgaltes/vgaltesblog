---
title: 'Deploying Azure Functions using slots'
date: '2018-11-18'
categories:
- serverless
tags:
- devops
- azure functions
- azure
---

In previous articles we've seen how to deploy a Function App using [Azure Functions Core Tools](http://vgaltes.com/post/deploying-azure-functions-an-introduction/), [Terraform](http://vgaltes.com/post/deploying-azure-functions-using-terraform/) and [Azure CLI with Zip Deploy](http://vgaltes.com/post/deploying-azure-functions-using-zip-deploy/). In this article we're going to take a look on how to deploy Function apps using [deployment slots](https://blogs.msdn.microsoft.com/appserviceteam/2017/06/13/deployment-slots-preview-for-azure-functions/).

## Why deployment slots
Being able to deploy into slots and decide which slot you use in production has many benefits. To name some of them:
 - Enable A/B testing
 - Enable blue/green deployments
 - No downtime deployments
 - Easy rollback system

So, let's take a look on how can we use them in Function apps.

## Creating a deployment slot
Deployment slots are not available in Azure Function Core Tools and it seems is not going to be in a [near future](https://github.com/Azure/azure-functions-core-tools/issues/227). So, we'll use Terraform and the Azure CLI to deal with them.

### Using terraform
Let's start using Terraform. We'll a slightly different version of the code we ended up doing in the [previous article](http://vgaltes.com/post/deploying-azure-functions-using-terraform/). I've just added a local variable to specify the name of the service and use more generic names in the resources, so you can re-use the script more easily:
```
locals {
  service_name = "testSlotsTF"
}

resource "azurerm_resource_group" "service-resource-group" {
  name     = "rg-${local.service_name}"
  location = "westeurope"
}

resource "azurerm_storage_account" "service-storage-account" {
  name                     = "${lower(local.service_name)}sa"
  resource_group_name      = "${azurerm_resource_group.service-resource-group.name}"
  location                 = "${azurerm_resource_group.service-resource-group.location}"
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_app_service_plan" "service-service-plan" {
  name                = "sp-${local.service_name}"
  location            = "${azurerm_resource_group.service-resource-group.location}"
  resource_group_name = "${azurerm_resource_group.service-resource-group.name}"
  kind                = "FunctionApp"

    sku {
    tier = "Dynamic"
    size = "Y1"
  }
}

resource "azurerm_storage_container" "service-storage-container" {
  name                  = "function-releases"
  resource_group_name   = "${azurerm_resource_group.service-resource-group.name}"
  storage_account_name  = "${azurerm_storage_account.service-storage-account.name}"
  container_access_type = "private"
}

resource "azurerm_storage_blob" "service-storage-blob" {
  name = "functionapp.zip"

  resource_group_name    = "${azurerm_resource_group.service-resource-group.name}"
  storage_account_name   = "${azurerm_storage_account.service-storage-account.name}"
  storage_container_name = "${azurerm_storage_container.service-storage-container.name}"

  type   = "block"
  source = "./../functionapp.zip"
}

data "azurerm_storage_account_sas" "service-storage-account-sas" {
  connection_string = "${azurerm_storage_account.service-storage-account.primary_connection_string}"
  https_only        = false

  resource_types {
    service   = false
    container = false
    object    = true
  }

  services {
    blob  = true
    queue = false
    table = false
    file  = false
  }

  start  = "2018-03-21"
  expiry = "2028-03-21"

  permissions {
    read    = true
    write   = false
    delete  = false
    list    = false
    add     = false
    create  = false
    update  = false
    process = false
  }
}

resource "azurerm_function_app" "service-function-app" {
  name                      = "${local.service_name}"
  location                  = "${azurerm_resource_group.service-resource-group.location}"
  resource_group_name       = "${azurerm_resource_group.service-resource-group.name}"
  app_service_plan_id       = "${azurerm_app_service_plan.service-service-plan.id}"
  storage_connection_string = "${azurerm_storage_account.service-storage-account.primary_connection_string}"

  app_settings {
    HASH            = "${base64sha256(file("./../functionapp.zip"))}"
    WEBSITE_USE_ZIP = "https://${azurerm_storage_account.service-storage-account.name}.blob.core.windows.net/${azurerm_storage_container.service-storage-container.name}/${azurerm_storage_blob.service-storage-blob.name}${data.azurerm_storage_account_sas.service-storage-account-sas.sas}"
  }
}
```

What are we going to do is to create a function app with no functions inside yet, and then create a staging slot with some code there. So, let's start commenting out the `app_settings` block of the function app. Now, it's time to add the staging slot, this time with some functions inside:

```
resource "azurerm_app_service_slot" "service-slot-staging" {
  name                = "staging"
  app_service_name    = "${azurerm_function_app.service-function-app.name}"
  location            = "${azurerm_resource_group.service-resource-group.location}"
  resource_group_name = "${azurerm_resource_group.service-resource-group.name}"
  app_service_plan_id = "${azurerm_app_service_plan.service-service-plan.id}"

  app_settings {
    HASH            = "${base64sha256(file("./../functionapp.zip"))}"
    WEBSITE_USE_ZIP = "https://${azurerm_storage_account.service-storage-account.name}.blob.core.windows.net/${azurerm_storage_container.service-storage-container.name}/${azurerm_storage_blob.service-storage-blob.name}${data.azurerm_storage_account_sas.service-storage-account-sas.sas}"
  }
}
```

And that's all. If you `terraform apply` these changes you will see that you have a function app with no code, and a staging slot with a function there.

![Deploy Slot Terraform](/images/DeploySlotTF.png)

### Using Azure CLI
We can accomplish the same using the Azure CLI. The only caveat here is that you're not going to find the slots commands under the `functionapp` command but under the `webapp` command. So, what you need to create a slot is the following command:
```
az webapp deployment slot create --name <functionAppName> --resource-group <resourceGroupName> --slot <slotName>
```

To deploy the function into that slot, you can use the same Zip Deploy command we used befor, but adding the --slot option:
```
az webapp deployment source config-zip -g <resourceGroupName> -n <functionAppName> --src <path/to/source.zip> --slot <slotName>
```

### Important warning
DON'T CREATE A SLOT CALLED PRODUCTION!!! Production is the name of the "default slot" that Azure uses, so if we try to create a slot called production, the operation will fail. Even worse, if we do that with Terraform, when we rename or remove the slot from our script, Terraform will try to remove or rename the production slot and the operation will fail because we're not allowed to do that and we'll end up in an annoying state. In case you do this, you will need to manually remove the slot from `terraform.tfstate` file.

## Swapping slots
At this time we've created a staging slot, we deployed some code there, we checked that everything is correct and we'd like to promote this to the production slot.

### Using Terraform
You can use Terraform to swith slots. We'll, not directly to switch slots, but to say which slot should use into the production one. To do that, you can use the [active slot](https://www.terraform.io/docs/providers/azurerm/r/app_service_active_slot.html) resource. In our case, we could use:
```
resource "azurerm_app_service_active_slot" "service-active-slot" {
  resource_group_name   = "${azurerm_resource_group.service-resource-group.name}"
  app_service_name      = "${azurerm_function_app.service-function-app.name}"
  app_service_slot_name = "${azurerm_app_service_slot.service-slot-staging.name}"
}
```

To be honest, I'm not sure how to integrate the operation of setting the active slot using Terraform. Any ideas will be welcomed in the comments :-)

### Using Azure CLI
To swap between two slots we just need to run the following command:
```
az webapp deployment slot swap  -g <resourceGroupName> -n <functionAppName> --slot <sourceSlot> --target-slot <targetSlot>
```

If you want to promote a slot to production, the targetSlot name needs to be `production`.

Let's take a look at the portal to see if the operation has succeeded:

![Deploy Slot After Swap](/images/DeploySlotAfterSwap.png)

Perfect! Now we have our function in production and an empty staging slot.

## Summary
In this article we've seen how to create deployment slots using Terraform and the Azure CLI and how to swap them.
