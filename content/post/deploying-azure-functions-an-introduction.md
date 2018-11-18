---
title: 'Deploying Azure Functions, an introduction'
date: '2018-11-12'
categories:
- serverless
tags:
- devops
- azure functions
- azure
---

In the last few days, I've been tinkering with [Azure Functions](https://azure.microsoft.com/en-gb/services/functions/), reading the documentation a bit and doing a [Pluralsight course](https://app.pluralsight.com/library/courses/azure-functions-fundamentals). As it happens quite offten, these introductory courses use easy techniques to deploy the code, focusing on showing what you can do with the platform. Although obviously this has some value, I don't think it's a good idea because, at the end, it will be something that you won't be able to use in a serious test.

Therefore, one of the things I always try to do once I know the basics is try to set up a very basic "deployment pipeline". Well, more than a deployment pipeline, I try to discover a way to deploy the code independently of the machine that is running it, a way that I can use from my laptop or from a CI agent.

When I've been developing using [AWS Lambda](https://aws.amazon.com/lambda/), my tool of choice has been the [Serverless framework](https://serverless.com/). This framework is really good and I encourage you to give it a try. You have other options too, like [SAM](https://docs.aws.amazon.com/lambda/latest/dg/serverless_app.html) from AWS, or many more.

The options if you're developing using Azure are more limited (a lot more limited I'd say). You can still use the serverless framework, but this time I wanted to see which options we have when we're using only Microsoft tools.

So let's try to deploy a very basic Azure Function without clicking on any UI.

## Installing the pre-requisites
In order to manage Azure Functions from the command line, you need a couple of tools to be installed in your laptop. I'm using a Mac, so the instructions explained here will be for that platform, but it should be easy to find the instructions for other platforms.

First, we need to install the Azure CLI. Just type `brew install azure-cli`. Once this is installed, you can install the Azure Functions Core Tools. To do that you will need to install nodejs and then run this command: `npm install -g azure-functions-core-tools`. If you don't want to install it using nodejs, you can use brew as well: `brew install azure-functions-core-tools`. More info on how to install it [here](https://github.com/Azure/azure-functions-core-tools).

Next step is to authenticate the Azure CLI. You have a couple of options here, but if you want something that is not interactive and that you can use in a CI agent you'll need to use a service principal. You can see how to create a new service principal [here](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest) and how to login using it [here](https://docs.microsoft.com/en-us/cli/azure/authenticate-azure-cli?view=azure-cli-latest#sign-in-with-a-service-principal).

Once we're authenticated, it's time to create the Function App.

## Creating the Function App and the Azure Function
In Azure, functions live inside a Function App. The Function App is the container of your functions, and it's where you define the consumption plan, the storage account, the runtime and other properties. 

Create an empty folder in your hard disk, cd into it and run the following command: `func init --worker-runtime dotnet`. This will create the Function App using DotNet as the runtime.

Next step is to create the Azure Function. To do that, just type: `func function new --language C# --template "HTTP trigger" --name HelloWorld`

In this case, we're using C# as language and "HTTP trigger" as the template. If you want to see the different templates available for the different languages, type `func templates list`.

Once we have the function we'd like to deploy, it's time to deploy it.

## Deploying the Function app
We've created a new Function App and a new Function, so deploy them should be a matter of running one command, right? Well, not really. There are several things we need to create before being able to deploy them. At the end of the article we'll create a deployment script to just run one command to deploy everything, but at the moment, let's do this step by step.

### Create the resource group
The first step is to create the resource group. A resource group is a virtual container of assets. To create it you need to run this command:
`az group create --name <name> --location <region>`

where name is the name of the resource group you want to create and region is the location of it. In our case we're going to use `westeurope`.

### Create the storage account
The next thing we need to create is the storage account. This account is needed to store the function files amongst other things. To create it you need to run this command:

`az storage account create --name <name> --location <region> --resource-group <resource_group_name> --sku Standard_LRS`

where name is the name of the storage account we want to create (must be between 3 and 24 characters in length and use numbers and lower-case letters only), region is the location where we want to create the storage account, resource_group_name is the name of the resource group that we've created in the previous step and sku is the type of storage account you want to create. The possible values for SKU are Premium_LRS, Standard_GRS, Standard_LRS, Standard_RAGRS, Standard_ZRS, being Standard_RAGRS the default one. We've chosen Standard_LRS because if the cheapest one. You can find more information [here](https://docs.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest#az-storage-account-create) and [here](https://azure.microsoft.com/en-gb/pricing/details/storage/).

### Create the Function App
The next step is to create the Function App in Azure. To do that, run the following command:
`az functionapp create --name <name> --storage-account <storage_account_name> --consumption-plan-location <region> --resource-group <resource_group_name>`

where name is the name of the Function App, storage_account_name is the name of the storage account we've created on the previous step, region is the location where the Function App will be hosted and resource_group_name is the name of the resource group we've created on the first step. You can find the documentation of the command [here](https://docs.microsoft.com/en-us/cli/azure/functionapp?view=azure-cli-latest#az-functionapp-create).

### Deploying
And finally we're ready to deploy. To deploy the Function App you just need to run the following command:
`func azure functionapp publish <name>`

where name is the name of the Function App. This will build the solution, publish the Function App and upload the package into Azure.

## Creating a deployment script
You can run all the commands explained before safely, again and again, they are idempotent. If the resource already exists, they will not do anything. That makes easier to wrap them all into a deployment script which we can call with the right parameters. Doing that, we can easily emulate the serverless framework, when they use the stage to separate deployments inside the same account. In this way, we'll be able to have different resource groups, storage accounts and functions (and eventually other resources) for different stages and even for different users.

This is the script we can use:
```
#!/bin/bash
location=$2
stage=$3
serviceName=$1-$stage
resourceGroupName=$serviceName"-rg"
serviceNameToLower="$(echo $serviceName | tr '[:upper:]' '[:lower:]')"
storageAccountName="$(echo ${serviceNameToLower//"-"/""})"
az group create --name $resourceGroupName --location $location
az storage account create --name $storageAccountName --location $location --resource-group $resourceGroupName --sku Standard_LRS
az functionapp create --name $serviceName --storage-account $storageAccountName --consumption-plan-location westeurope --resource-group $resourceGroupName
func azure functionapp publish $serviceName
```

and we can call it in this way:
`./deploy.sh Test-Deploy-VGA westeurope dev`

The script will use the first parameter as the name of the service (the Function App) and will use it to create the names of the resource group and storage account. In the case of the storage account, it will convert the name to lower case and remove the hyphens. Then, it will call the commands we've described previously and will deploy the Function App.

## Summary
In this (introductory) article we've seen how we can publish easily a Function App into Azure. We've seen how we can create a simple script to help us run all the necessary steps and even create separate deployments for different stages.