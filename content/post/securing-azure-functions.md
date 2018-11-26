---
title: 'Securing Azure Functions'
date: '2018-11-23'
categories:
- serverless
tags:
- devops
- azure functions
- azure
---

(Thanks to Francesc Ribera, [Alex Casquete](https://acasquete.github.io/) and [Jerry Liu](https://stackoverflow.com/questions/53434852/securing-an-azure-function) for their guidance on this topic)

As far as I know (correct me in the comments if I'm wrong, please), when you create a brand new Azure Function, it has access to everything in your subscription as a contributor. That's really not great. When you develop a new Lambda function you have to specify all the permissions that the function has to have. For example, if you need to query and put items into a DynamoDB table (and you're using the serverless framework), you have to do something like this:

```
functions:
  watchStream:
    handler: src/functions/user/watchStream.handler
    events:
      - http:
          path: user/{user_id}/stream
          method: post
    environment:
      videoStreamsTableName: ${self:custom.videoStreamsTableName}
    iamRoleStatements:
      - Effect: Allow
        Action:
          - dynamodb:Query
          - dynamodb:PutItem
        Resource: "arn:aws:dynamodb:#{AWS::Region}:#{AWS::AccountId}:table/${self:custom.videoStreamsTableName}"
```

This is great for many reasons. By default you function doesn't have access to anything, which makes easier to control it. The level of detail that you get is great, as you've seen, you can define permissions at a table level. And this way of assigning permissions makes easier to follow the [Least Privilege Principle](). 

But it looks like that the Role Base Security in Azure is still on preview and not available in all the regions, so we need to use some other system to try to access the permissions of our functions.

## Shared Access Policy to the rescue
The Shared Access Policy is a way to grant access to our resources by adding a signature to the connection string. This will tell the resource which operations to accept. In this example we'll try to write to an Event Hub from a function. We'll do this using the .Net client library and an output binding.

## Creating the Event Hub
We need to update our deployment script with the following content:
```
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
az functionapp config appsettings set -g $resourceGroupName -n $serviceName --settings TestQueueConnectionString=$ehConnectionString
```

First, we need to create the namespace, which is where we'll specify the pricing tier, in this case Basic. Then, we create the event hub. Here, we need to specify the message retention value, because the defuault is 7 and that's a value not supporting in the Basic princing tier. The last step is to create the shared access policy. In this case, we say that we want to call it `Send` and that it will have just `Send` rights. 

## Writing to the event hub using the .Net library
Let's start writing a message using the .Net library. First of all, add the necessary package:
```
dotnet add package Microsoft.Azure.EventHubs
```

Import the package

```
using Microsoft.Azure.EventHubs;
```

Get the configuration setting (in the V2 of Azure Functions we need to use the new [ConfigurationBuilder](https://blog.jongallant.com/2018/01/azure-function-config/) class):

```
var config = new ConfigurationBuilder()
        .SetBasePath(context.FunctionAppDirectory)
        .AddJsonFile("local.settings.json", optional: true, reloadOnChange: true)
        .AddEnvironmentVariables()
        .Build();
```

Get the connection string:
```
var ehConnectionString = config["TestQueueConnectionString"];
```

And finally write a message into the event hub:
```
var ehClient = EventHubClient.CreateFromConnectionString(ehConnectionString);
      await ehClient.SendAsync(new EventData(Encoding.UTF8.GetBytes("This is a message")));
      await ehClient.CloseAsync();
```

Now it's time to deploy and test our function. As always, run the deployment script: 
```
./deploy.sh TestAuth westeurope dev
```

If we now go to the event hub we'll see that we've received a message

![A message](/images/MessageReceived.png)

Let's see what happens if we change the policy. Instead of Read as right, change it to Listen and redeploy. Call the function and you'll see that you're not getting anything back. That's because you've got an error. If you check the status code you'll see that you've got a 500.

Go to your function monitor and you'll see that you've got an `Unauthorised access` error.

![Unauthorised Access](/images/UnauthorisedAccess.png)

## Writing to the event hub using an output binding
An [output binding](https://docs.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings) is a declarative way to connect to data from our code. I still haven't worked in anger with them, but I have mixed feelings about them. My first impression is that it's too much magic. But let's see and example of how to use them.

First of all, add the necessary package and import it:

```
dotnet add package Microsoft.Azure.WebJobs.Extensions.EventHubs
```

```
using Microsoft.Azure.WebJobs;
```

Change the signature of the function to something like:

```
public static ActionResult Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] CustomObject req,
        [EventHub("pings", Connection = "TestQueueConnectionString")] out string ehMessage,
        ILogger log)
    {
```

And finally write the message to the event hub by just writing it into the out parameter:

```
ehMessage = "Message from output binding";
```

Let's deploy and test the function. You should see a new message in the event hub (remember to change the configuration of the Shared Access Policy to Send)

You should see a 200 response..

![Event Hub binding](/images/EventHubBinding.png)

and a new message in the event hub

![Event Hub binding message](/images/EventHubBindingMessage.png)

## Imperative binding
One of the problems with this approach is that we're storing the connection string in the application settings, which is not a [great](https://jan-v.nl/post/working-with-azure-key-vault-in-azure-functions) idea. Fortunately, we have another option, which are the imperative bindings. Obviously, we can achieve the same results using directly the .Net library. 

So, let's say that we don't want to store the connection string in the application settings. We should store them in something like Key Vault, or Hashicorp Vault, or whatever your organization has. The problem is that we can't use that with a normal binding, so we need to use imperative bindings.

Note: we're going to explain how to use Key Vault in another article. For now, imagine that we have the connection string in a variable called ehConnectionString.

First, we need to change the signature of the function to include the Binder object:
```
[FunctionName("HelloWorld")]
    public async static Task<ActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] CustomObject req,
        ExecutionContext context,
        Binder binder,
        ILogger log)
    {
```

Then, we need to update the get the settings object:
```
var config = new ConfigurationBuilder()
        .SetBasePath(context.FunctionAppDirectory)
        .AddJsonFile("local.settings.json", optional: true, reloadOnChange: true)
        .AddEnvironmentVariables()
        .Build();
```

Now, we need to update the setting the binder is going to use:
```
config["TestQueueConnectionString"] = ehConnectionString;
```

It's time to create the binding:
```
var output = await binder.BindAsync<IAsyncCollector<string>>(new EventHubAttribute("pings")
    {
      Connection = "TestQueueConnectionString"
    });
```

Note that (I don't know why) we can't use directly a string with the binder, we need to use a collector or an async collector.

And finally, add the message:
```
await output.AddAsync("A new message using imperative binding.");
```

## Summary
In this article we've seen how to limit what an Azure Function can access. We've implemented that using Shared Access Policies and we've seen an example with an Event Hub. We've also learned how to deploy an event hub using the Azure CLI. We learned how to publish a message into an event hub using the .Net library, an output binding and an imperative binding.
