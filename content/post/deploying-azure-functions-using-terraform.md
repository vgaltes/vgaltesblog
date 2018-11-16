---
title: 'Deploying Azure Functions using Terraform'
date: '2018-11-16'
categories:
- serverless
tags:
- devops
- azure functions
- azure
---

In previous articles ([I](https://vgaltes.com/post/deploying-azure-functions-an-introduction/), [II](https://vgaltes.com/post/deploying-azure-functions-using-cirecleci/)) we've seen how to deploy an Azure Function App using the Azure CLI and the Azure Functions Core Tools. In this article we'll see how to deploy it using [Terraform](https://www.terraform.io/).

## Prerequisites
In order to follow this article you will need the .Net SDK 2.1, the Azure CLI and Terraform installed in your laptop/container/VM/whatever.

## Building the package
What Terraform is going to do is take advantage of the [Zip deployment](https://docs.microsoft.com/en-us/azure/azure-functions/deployment-zip-push) capability (More on this in a future article). So, what we need to do first is to create a zip with the contents we'd like to deploy. We'll call this zip `functionapp.zip` To do that, you can use this script:

```
dotnet publish
pushd bin/Debug/netcoreapp2.1/publish
zip -r -X ../../../../functionapp.zip *
popd
```

## Create the Terraform script
Let's start creating the terraform script. Create a folder in your project called `infrastructure` and create file called, for example, `functionapp.tf`. Open the file to edit its contents.

The first thing we'll need to do is specify the [Resource Group](https://www.terraform.io/docs/providers/azurerm/d/resource_group.html) we'd like to create:
```
resource "azurerm_resource_group" "rg-testDeployTF" {
  name     = "rg-testDeployTF"
  location = "westeurope"
}
```

Then, we need to specify the [Storage Account](https://www.terraform.io/docs/providers/azurerm/r/storage_account.html):
```
resource "azurerm_storage_account" "sa-testDeployTF" {
  name                     = "testdeploytfsa"
  resource_group_name      = "${azurerm_resource_group.rg-testDeployTF.name}"
  location                 = "${azurerm_resource_group.rg-testDeployTF.location}"
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```

It's time to specify the [Service Plan](https://www.terraform.io/docs/providers/azurerm/d/app_service_plan.html):
```
resource "azurerm_app_service_plan" "sp-testDeployTF" {
  name                = "sp-testDeployTF"
  location            = "${azurerm_resource_group.rg-testDeployTF.location}"
  resource_group_name = "${azurerm_resource_group.rg-testDeployTF.name}"
  kind                = "FunctionApp"

  sku {
    tier = "Dynamic"
    size = "Y1"
  }
}
```

Now it's time to update the recently created zip file into a [blob](https://www.terraform.io/docs/providers/azurerm/r/storage_blob.html). We'd need to create a blob inside a [storage container](https://www.terraform.io/docs/providers/azurerm/r/storage_container.html):
```
resource "azurerm_storage_container" "sc-testDeployTF" {
  name                  = "function-releases"
  resource_group_name   = "${azurerm_resource_group.rg-testDeployTF.name}"
  storage_account_name  = "${azurerm_storage_account.sa-testDeployTF.name}"
  container_access_type = "private"
}

resource "azurerm_storage_blob" "sb-testDeployTF" {
  name = "functionapp.zip"

  resource_group_name    = "${azurerm_resource_group.rg-testDeployTF.name}"
  storage_account_name   = "${azurerm_storage_account.sa-testDeployTF.name}"
  storage_container_name = "${azurerm_storage_container.sc-testDeployTF.name}"

  type   = "block"
  source = "./../functionapp.zip"
}
```

In order that the Function App has access to this blob, we need to create a [Shared Access Signature](https://www.terraform.io/docs/providers/azurerm/d/storage_account_sas.html):
```
data "azurerm_storage_account_sas" "sas-testDeployTF" {
  connection_string = "${azurerm_storage_account.sa-testDeployTF.primary_connection_string}"
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
```

And finally we need to create the Function App and tell it that it needs to take the code from the uploaded zip file:
```
resource "azurerm_function_app" "testDeployTF" {
  name                      = "testDeployTF"
  location                  = "${azurerm_resource_group.rg-testDeployTF.location}"
  resource_group_name       = "${azurerm_resource_group.rg-testDeployTF.name}"
  app_service_plan_id       = "${azurerm_app_service_plan.sp-testDeployTF.id}"
  storage_connection_string = "${azurerm_storage_account.sa-testDeployTF.primary_connection_string}"

  app_settings {
    HASH            = "${base64sha256(file("./../functionapp.zip"))}"
    WEBSITE_USE_ZIP = "https://${azurerm_storage_account.sa-testDeployTF.name}.blob.core.windows.net/${azurerm_storage_container.sc-testDeployTF.name}/${azurerm_storage_blob.sb-testDeployTF.name}${data.azurerm_storage_account_sas.sas-testDeployTF.sas}"
  }
}
```

## Deploying

It's time to cd into the infrastructure folder, login into azure using `az login` and type `terraform apply`.

![Azure Function Terraform](/images/AzureFunctionTerraform.png)

Here it is! The Function App deployed into Azure :-)

## Summary
In this article we've seen how we can deploy a Function App using Terraform.
