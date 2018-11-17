---
title: 'Deploying Azure Functions using Zip Deploy'
date: '2018-11-17'
categories:
- serverless
tags:
- devops
- azure functions
- azure
---

In a [previous article](http://vgaltes.com/post/deploying-azure-functions-an-introduction/) we've seen how to deploy a Function App using the Azure CLI and the Azure Functions Core Tools. In this article we'll see how to get rid of the help of the latter and use the [Zip Deploy](https://docs.microsoft.com/en-us/azure/azure-functions/deployment-zip-push) feauture.

## Fixing the script.
We already have a deployment script. It is something like this:
```
#!/bin/bash
location=$2
stage=$3
serviceName=$1-$stage
resourceGroupName=$serviceName"-rg"
serviceNameToLower="$(echo $serviceName | tr '[:upper:]' '[:lower:]')"
storageAccountName="$(echo ${serviceNameToLower//"-"/""})"
echo "Creating resource group"
az group create --name $resourceGroupName --location $location
echo "Creating storage account"
az storage account create --name $storageAccountName --location $location --resource-group $resourceGroupName --sku Standard_LRS
echo "Creating function app"
az functionapp create --name $serviceName --storage-account $storageAccountName --consumption-plan-location westeurope --resource-group $resourceGroupName
echo "Publishing function locally"
dotnet build
echo "Publisihng function to Azure"
func azure functionapp publish $serviceName
```

What are we going to do is to change the last two steps.

### Publshing the function locally
Now, what we'll need to do is to create a zip from the result of publishing the project. 
```
echo "Publishing function locally"
mkdir publish
rm -rf publish/*
dotnet publish
pushd bin/Debug/netcoreapp2.1/publish
zip -r -X ../../../../publish/$serviceName.zip *
popd
```

### Publishing the Function App to Azure
And finally we need to use the Azure CLI to push that zip.
```
echo "Publisihng function to Azure"
az webapp deployment source config-zip -g $resourceGroupName -n $serviceName --src ./publish/$serviceName.zip
```

## Testing
Let's call the script and see if it works:
` ./deploy.sh testZipDeploy westeurope uat`

![Zip deployment](images/ZipDeploy.png)

Here it is!

## CI implications
We've simplified how to deploy the Function App. If you've followed this [article](http://vgaltes.com/post/deploying-azure-functions-using-cirecleci/) you will remember that we had to install the Azure Functions Core Tools in the docker image. We no longer need this. All the other steps will be the same.


## Summary
In this article we've seen how to deploy a Function App using Zip Deploy so we don't need the help of the Azure Function Core Tools.
