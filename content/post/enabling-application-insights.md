---
title: 'Deploying Azure Functions - Enabling Application Insights'
date: '2018-11-20'
categories:
- serverless
tags:
- devops
- azure functions
- azure
---

In our previous articles we've seen how to deploy a Function App using the [Azure CLI](http://vgaltes.com/post/deploying-azure-functions-using-zip-deploy/) or [Terraform](http://vgaltes.com/post/deploying-azure-functions-using-terraform/). Once we have a function deployed and accepting traffic, the next obvious thing to do is to monitor it.

Adding some logs to the function is fairly easy. If you take a look at the signature of the function
```
[FunctionName("HelloWorld")]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req,
        ILogger log)
```

you'll see that we hava a parameter of type `ILogger` that we can use to, yes, you guessed it right, log. The object has some helper methods that will allow you to log using different log levels and it's [quite easy](https://docs.microsoft.com/en-us/azure/azure-functions/functions-monitoring#structured-logging) to use structured logging too.

But, where can I see those logs?

## Azure storage
By default, logs go to the Azure Storage. You can find it under your subscription -> <storage_account_name> -> File Shares -> Function App name -> LogFiles -> application -> functions -> function -> <function_name>. Easy to remember, isn't it?

You can download the log file if you want. As far as I know, there's no easy way to search in those log files. Apart from that, when you develop a serverless application, you will have more than function, so you will need to search amongst different log files, which will make things more complicated.

## Application insights
The proposed solution by Microsoft is to use Application Insights. Application Insights will give us much more information and will allow us to query our logs in a much easier way. Application insights is not enabled by default, so we will need to enable it.

### Enabling using Azure CLI
First we'll take a look on how to do this via the Azure CLI. We'll include this in our deployment script. First we need to create the Application Insights resource:
```
az resource create --resource-group $resourceGroupName --resource-type "Microsoft.Insights/components" --name $appInsightsName --location $location --properties '{"ApplicationType":"web"}'
```

next we need to get the instrumentation key, that we will need to add to the application settings of the Function App:
```
appInsightsInstrumentationKey="$(az resource show -g $resourceGroupName -n $appInsightsName --resource-type 'Microsoft.Insights/components' --query properties.InstrumentationKey)"
```

this instrumentation key comes with quotes, which we need to remove:
```
appInsightsInstrumentationKey="${appInsightsInstrumentationKey%\"}"
appInsightsInstrumentationKey="${appInsightsInstrumentationKey#\"}"
```

and finally, after the Function App is created, we need to add the instrumentation key to the application settings:
```
az functionapp config appsettings set -g $resourceGroupName -n $serviceName --settings APPINSIGHTS_INSTRUMENTATIONKEY=$appInsightsInstrumentationKey
```

### Enabling using Terraform
We can achieve the same results via Terraform. First, we need to create the Application Insights resource:
```
resource "azurerm_application_insights" "service-application-insights" {
  name                = "ai-${local.service_name}"
  location            = "${azurerm_resource_group.service-resource-group.location}"
  resource_group_name = "${azurerm_resource_group.service-resource-group.name}"
  application_type    = "Web"
}
```

and then we need to set up the application setting when we create the Function App:
```
resource "azurerm_function_app" "service-function-app" {
  name                      = "${local.service_name}"
  location                  = "${azurerm_resource_group.service-resource-group.location}"
  resource_group_name       = "${azurerm_resource_group.service-resource-group.name}"
  app_service_plan_id       = "${azurerm_app_service_plan.service-service-plan.id}"
  storage_connection_string = "${azurerm_storage_account.service-storage-account.primary_connection_string}"

  app_settings {
    APPINSIGHTS_INSTRUMENTATIONKEY = "${azurerm_application_insights.service-application-insights.instrumentation_key}"
  }
}
```

And that's all we need. We now have Application Insights enabled in our Function App.

## Summary
In this article we've seen how to enable Application Insights in our Function App, via Azure CLI and Terraform.

Observability it's a broader topic that just logging. Application insights should help there quite a lot. Another thing to consider is how to handle correlation ids in our application, and how make that those correlation ids make sense with all the information that Application Insights give us. What we want is that, using a simple query, know what happened to an invocation of a service. We'll talk about that hopefully in a future post.
