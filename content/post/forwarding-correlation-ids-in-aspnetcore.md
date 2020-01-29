---
title: 'Forwarding correlation IDs in ASPNet Core'
date: '2020-01-29'
categories:
- aspnet core
tags:
- microservices
---

When you have different services that communicate amongst them, you need to be able to correlate those calls to perform effective analysis of any problem you might have. The way to do this is using correlation ids and pass them along in all your call to the different services you use. In this article, I´m going to explain you a way to do this in ASP .Net Core.

## Correlation IDs
Why do you need more than one correlation ID? As Yan Cui explains in this [article](https://theburningmonk.com/2015/05/a-consistent-approach-to-track-correlation-ids-through-microservices/) you might be interested in passing other data between your services, not just the correlation ID. Things like the user ID, or the session ID or maybe things related to your domain as order ID.

## Storing the correlation IDs to enable its use in other parts of the application
First of all, we'll implement a simple class to update and store the correlation IDs

```
public class CorrelationIDs
{
    private readonly Dictionary<string, string> currentIDs = new Dictionary<string, string>();
    private const string CorrelationIdKey = "x-correlation-id";

    public CorrelationIDs()
    {
        currentIDs[CorrelationIdKey] = Guid.NewGuid().ToString();
    }

    public IReadOnlyDictionary<string, object> GetCurrentIDs()
    {
        return currentIDs.ToDictionary(k => k.Key, k => (object)k.Value);
    }

    public void Update(string key, string value)
    {
        currentIDs[key] = value;
    }
}

```

As you can see it's a quite simple class in which we're defining that "x-correlation-id" will be a required id and, for that reason, we're initialising it.

We need to register this class, so we're adding it as scoped:
```
services.AddScope<CorrelationIDs, CorrelationIDs>();
```

## Reading the correlation IDs
In this example, we're going to suppose that the communication between the services uses HTTP calls. So, we'll need to read the correlation IDs from the request headers. We can do this using a middleware that inspects the headers and uses the previous class to store them. Something like this:

```
public class LogContextMiddleware
{
    private readonly RequestDelegate next;
    private readonly ILogger<LogContextMiddleware> logger;

    public LogContextMiddleware(RequestDelegate next, ILogger<LogContextMiddleware> logger)
    {
        this.next = next;
        this.logger = logger;
    }

    public Task InvokeAsync(HttpContext context, CorrelationIDs correlationIDs)
    {
        var correlationHeaders = context.Request.Headers
            .Where(h => h.Key.ToLowerInvariant().StartsWith("x-correlation-"))
            .ToDictionary(h => h.Key, h => (object)h.Value.ToString());
        
        foreach (var correlationHeader in correlationHeaders)
        {
            correlationIDs.Update(correlationHeader.Key, correlationHeader.Value.ToString());
        }
    }
}
```

As you can see, it takes all the headers that start with "x-correlation-" and stores them using the CorrelationIDs class.

## Writing the correlation IDs in the logs
The next step is to write the correlation IDs in any log line we produce. We can use the method BeginScope of the ILogger interface to do this. So, we need to add the following lines to the end of the middleware we just wrote.

```
// ensures all entries are tagged with some values
using (logger.BeginScope(correlationIDs.GetCurrentIDs()))
{
    // Call the next delegate/middleware in the pipeline
    return next(context);
}
```

This approximation has the advantage that is agnostic to the library you use to log.

## Sending the correlation IDs to the other services.
The last step would be to send the correlation IDs we just captured to the other services. To communicate with other services, we're going to use Typed Clients. You can read about the "new" features regarding HttpClients in this excellent [series of posts](https://www.stevejgordon.co.uk/introduction-to-httpclientfactory-aspnetcore) from [Steve Gordon](https://twitter.com/stevejgordon).

Let's start writing a base class that will be used by our Typed Clients:

```
public abstract class ApiClient
{
    private readonly HttpClient httpClient;
    private readonly CorrelationIDs correlationIDs;

    protected GenericApiClient(HttpClient httpClient, CorrelationIDs correlationIDs)
    {
        this.httpClient = httpClient;
        this.correlationIDs = correlationIDs;
    }
    public async Task<HttpResponseMessage> SendAsync(HttpMethod method, string uri)
    {
        var request = new HttpRequestMessage
        {
            RequestUri = new Uri(httpClient.BaseAddress + uri),
            Method = method
        };

        AddCorrelationHeaders(request, correlationIDs);

        return await httpClient.SendAsync(request);
    }

    private void AddCorrelationHeaders(HttpRequestMessage request, CorrelationIDs correlationIDs)
    {
        foreach (var (key, value) in correlationIDs.GetCurrentIDs())
        {
            request.Headers.Add(key, value.ToString());
        }
    }
}
```

The interesting thing regarding correlation IDs is in the AddCorrelationHeaders method, where we're setting the headers for the call.

Next, we need to create the Type Client for our specific API. The API I'm using a very simple API with just one method, so the Client would be something like this:
```
public class ValuesApiClient : ApiClient, IValuesApiClient
{
    private readonly ILogger<ValuesApiClient> logger;

    public ValuesApiClient(HttpClient httpClient, CorrelationIDs correlationIDs, ILogger<ValuesApiClient> logger) : base(httpClient, correlationIDs)
    {
        this.logger = logger;
    }

    public async Task<ValueItem> GetValueItem(string key)
    {
        logger.LogInformation("Calling Values Api. {Key}", key);

        var response = await SendAsync(HttpMethod.Get, $"api/values/{key}");
        var json = await response.Content.ReadAsStringAsync();
        var result = JsonConvert.DeserializeObject<ValueItem>(json);

        return result;
    }
}
```

Nothing special here. 

Finally, we need to register the Typed Client. We can do that in the `ConfigureServices` method the `Startup.cs` class.

```
.AddHttpClient<IValuesApiClient, ValuesApiClient>(client =>
{
    client.BaseAddress = new Uri(apisSettings.ValueItemsApi);
    client.DefaultRequestHeaders.Add("Accept", "application/json");
    client.DefaultRequestHeaders.Add("User-Agent", "ValuesWebApi");
})
```

## Summary
In this article, we've seen how we capture and forward correlation IDs for HTTP APIs in ASPNet Core. Hope it helps!!
