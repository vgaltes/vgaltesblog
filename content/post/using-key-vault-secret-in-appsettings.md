---
title: 'Using Key Vault secrets in AppSettings'
date: '2018-11-30'
categories:
- serverless
tags:
- devops
- azure functions
- azure
---

In the [last article](http://vgaltes.com/post/securing-azure-functions/) we talked about securing Azure Functions and we saw how to insert a message into an Event Hub. To insert the message, we needed the connection string to be in an application setting. This is not the most secure way to store a connection string. We talked that it would be a much better option to store it in Key Vault. Until a couple of days ago, to do that we needed to use a library to get the secret from Key Vault and then use an imperative binding to be able to insert the message into the Event Hub.

But on Wednesday the Azure Functions team released a bunch of [security related features](https://azure.microsoft.com/en-us/blog/simplifying-security-for-serverless-and-web-apps-with-azure-functions-and-app-service/). One of them, is that you can use a secret stored in Key Vault as a value for an application settings. So, we can use Key Vault as the source of our secrets for the declarative bindings without making any change in our code. Let's see how we can do it.

## Creating the Key Vault
As usual, we can do this using the Azure Portal, but we don't really want to use it, so we'll continue using the Azure CLI. Open your preferred terminal and type:
```
keyVaultName=$serviceName-kv
az keyvault create --name $keyVaultName --resource-group $resourceGroupName --location $location
```

Next, we need to create a key to encrypt all our passwords. By now we'll tell Key Vault to create it for us.
```
keyVaultKeyName=$keyVaultName-key
az keyvault key create --vault-name $keyVaultName --name $keyVaultKeyName --protection software
```

The next thing we need to know is to create the Service Managed Identity of our Function App. We will need it to be able to set a police on the Key Vault so our function can read secrets from there. Let's do this, step by step. First, create the SMI and get the service principal id:

```
spn="$(az webapp identity assign --name $serviceName --resource-group $resourceGroupName --query principalId)"
spn="${spn%\"}"
spn="${spn#\"}"
```

And second, create the policy in the Key Vault:

```
az keyvault set-policy --name $keyVaultName --object-id $spn --secret-permissions get
```

We now need to create the secret and get its id:

```
secretId="$(az keyvault secret set --name EventHubConnectionString --vault-name $keyVaultName --value $ehConnectionString --query id)"
secretId="${secretId%\"}"
secretId="${secretId#\"}"
```

And finally we need to set the application setting in our function. The setting has to have this format: `@Microsoft.KeyVault(SecretUri=<the_uri_of_the_secret_including_the_version>)`

```
ehConnectionString="@Microsoft.KeyVault(SecretUri=$secretId)"
az functionapp config appsettings set -g $resourceGroupName -n $serviceName --settings TestQueueConnectionString=$ehConnectionString

```

And now we can use this setting in the biding of our function, the same way we used it in the first example of the previous post:

```
[FunctionName("HelloWorld")]
public static ActionResult Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] CustomObject req,
    [EventHub("pings", Connection = "TestQueueConnectionString")] out string ehMessage,
    ILogger log)
{
  string name = req?.name;

  ehMessage = "Message from output binding from key vault";

  return name != null
      ? (ActionResult)new OkObjectResult($"Hello again, this time with custom request object, {name}")
      : new BadRequestObjectResult("Please pass a name on the query string or in the request body");
}
```

And that's all. This is how we used a secret stored in Key Vault in a declarative binding.

For the record, this is how our deploy.sh script looks like:

```
#!/bin/bash
location=$2
stage=$3
serviceName=$1-$stage
BLUE='\033[0;34m'
NC='\033[0m' # No Color

resourceGroupName=$serviceName"-rg"
appInsightsName=$serviceName"-ai"
serviceNameToLower="$(echo $serviceName | tr '[:upper:]' '[:lower:]')"
storageAccountName="$(echo ${serviceNameToLower//"-"/""})"
echo -e "${BLUE}Creating resource group${NC}"
az group create --name $resourceGroupName --location $location
echo -e "${BLUE}Creating storage account${NC}"
az storage account create --name $storageAccountName --location $location --resource-group $resourceGroupName --sku Standard_LRS
echo -e "${BLUE}Creating function app${NC}"
az functionapp create --name $serviceName --storage-account $storageAccountName --consumption-plan-location $location --resource-group $resourceGroupName
echo -e "${BLUE}Creating the app insights resource${NC}"
az resource create --resource-group $resourceGroupName --resource-type "Microsoft.Insights/components" --name $appInsightsName --location $location --properties '{"ApplicationType":"web"}'
appInsightsInstrumentationKey="$(az resource show -g $resourceGroupName -n $appInsightsName --resource-type 'Microsoft.Insights/components' --query properties.InstrumentationKey)"
appInsightsInstrumentationKey="${appInsightsInstrumentationKey%\"}"
appInsightsInstrumentationKey="${appInsightsInstrumentationKey#\"}"
echo -e "${BLUE}Instrumentation key = $appInsightsInstrumentationKey${NC}"
echo -e "${BLUE}Publishing function locally${NC}"
mkdir publish
rm -rf publish/*
dotnet publish
pushd bin/Debug/netcoreapp2.1/publish
zip -r -X ../../../../publish/$serviceName.zip *
popd
echo -e "${BLUE}Publisihng function to Azure${NC}"
az webapp deployment source config-zip -g $resourceGroupName -n $serviceName --src ./publish/$serviceName.zip
#config app settings
echo -e "${BLUE}Configuring the instrumentation key${NC}"
az functionapp config appsettings set -g $resourceGroupName -n $serviceName --settings APPINSIGHTS_INSTRUMENTATIONKEY=$appInsightsInstrumentationKey

#Create the event hub
echo -e "${BLUE}Creating the Event Hub${NC}"
eventHubNamespaceName=$serviceName"-ehnamespace"
eventHubName=pings
az eventhubs namespace create --name $eventHubNamespaceName --resource-group $resourceGroupName --location $location --sku Basic
az eventhubs eventhub create --name $eventHubName --namespace-name $eventHubNamespaceName --resource-group $resourceGroupName --message-retention 1
az eventhubs eventhub authorization-rule create --resource-group $resourceGroupName --namespace-name $eventHubNamespaceName --eventhub-name $eventHubName --name Send --rights Send
ehConnectionString="$(az eventhubs eventhub authorization-rule keys list --resource-group $resourceGroupName --namespace-name $eventHubNamespaceName --eventhub-name $eventHubName --name Send --query primaryConnectionString)"
ehConnectionString="${ehConnectionString%\"}"
ehConnectionString="${ehConnectionString#\"}"

#Create the Key Vault
echo -e "${BLUE}Creating the Key vault${NC}"
keyVaultName=$serviceName-kv
keyVaultKeyName=$keyVaultName-key
az keyvault create --name $keyVaultName --resource-group $resourceGroupName --location $location
az keyvault key create --vault-name $keyVaultName --name $keyVaultKeyName --protection software
spn="$(az webapp identity assign --name $serviceName --resource-group $resourceGroupName --query principalId)"
spn="${spn%\"}"
spn="${spn#\"}"
az keyvault set-policy --name $keyVaultName --object-id $spn --secret-permissions get
secretId="$(az keyvault secret set --name EventHubConnectionString --vault-name $keyVaultName --value $ehConnectionString --query id)"
secretId="${secretId%\"}"
secretId="${secretId#\"}"
ehConnectionString="@Microsoft.KeyVault(SecretUri=$secretId)"
az functionapp config appsettings set -g $resourceGroupName -n $serviceName --settings TestQueueConnectionString=$ehConnectionString
```

You can call it in this way:
```
./deploy.sh TestKeyVault westeurope dev
```

## Summary
In this article we've seen how we can use one of the new security related features of Azure Functions and use a secret stored in Key Vault in a declarative binding. I think it's great news if you use this kind of bindings. Still has room to improve, like that you have to put the version of the secret, but they are working on it. In my opinion, although it will need code changes in your function, it would be better if you can say in the binding that the data come from Key Vault, instead of going through the indirection of the application settings. Maybe something like:
```
FunctionName("HelloWorld")]
public static ActionResult Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] CustomObject req,
    [EventHub("pings", Connection = "EventHubConnectionString" VaultName = "TestKeyVault-kv")] out string ehMessage,
    ILogger log)
{
```

What do you think?