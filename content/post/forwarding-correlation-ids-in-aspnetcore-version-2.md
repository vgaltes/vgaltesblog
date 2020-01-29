---
title: 'Capturing and forwarding correlation IDs in ASP.NET Core, the easy way'
date: '2020-01-30'
thumbnail: /images/ActivityBaggage.png
categories:
- aspnet core
tags:
- microservices
---

The minute after I published my last article about [Capturing and forwarding correlation IDs](https://vgaltes.com/post/forwarding-correlation-ids-in-aspnetcore/), my very good friend [Hugo Biarge](https://twitter.com/hbiarge) send me a Direct Message telling me: "Hey man! Have you read this [article](https://devblogs.microsoft.com/aspnet/improvements-in-net-core-3-0-for-troubleshooting-and-monitoring-distributed-apps/)? This is new from  ASP .NET Core 3, and it's an easier solution than the one you explain in the article." So, I took a look, not only at the article but also at the traces that I was already generating, and voilà, everything was already there. Let me explain this a bit better.

ASP .NET Core 3 has correlated logs "out of the box" by following the proposed [trace context standard](https://www.w3.org/TR/trace-context/). Remember the demo we set up in the previous article, let's explore the logs it generates before changing anything. If we take a look at the logs, we'll see that every log trace has a property called TraceId (created "automagically") and that this TraceId is the same for all the traces of a single request, that, in our case, includes an "external" API making calls to an "internal" API. This is amazing! Without having to add a single line of code, we have distributed tracing implemented!

"Yeah, that's cool, but in your previous example, you set the correlation ID using a header when calling the external API. Can you do that here?" 

Good question! You can do that by setting a header called traceparent (all lowercase). Whatever you set in that header, it goes to the TraceId field.

"Nice! But you also could send other correlation IDs apart from the TraceId. How do you do that now?"

Another good question! But the answer is even better: you don't have to do anything! Let's make a demo: put a breakpoint in both controllers and then make a call with the header [Correlation-Context](https://github.com/w3c/correlation-context/blob/master/correlation_context/HTTP_HEADER_FORMAT.md) set. An example of the header could be: `Correlation-Context: userId=sergey,serverNode=DF,isProduction=false`

Nice, let's make the call and in both controllers inspect the `System.Diagnostics.Activity.Baggage`

![System.Diagnostics.Activity.Baggage](/images/ActivityBaggage.png)

Voilà! Both properties are there in both controllers! That's super cool! We haven't touched a single line of code, and our correlation information is traveling through all our services.

"I see. This is really cool. But I'm inspecting the traces, and I can't see any of that correlation information, just the TraceId (and ParentId and SpanId)."

Yes, you're right, but that's the only point where we have to add some code. Fortunately, half of the code is already there. Remember the LogContextMiddleware we talked about in the last article? The one that reads the headers updates the list of correlation IDs, and sets the context of the logger with those values? We just need to throw most of its code and get the correlation IDs from the `Activity.Baggage` object.

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
    
    public Task InvokeAsync(HttpContext context)	 
    {	 
        var correlationHeaders = Activity.Current.Baggage.ToDictionary(b => b.Key, b => (object) b.Value);	 
   
        // ensures all entries are tagged with some values	 
        using (logger.BeginScope(correlationHeaders))	 
        {	 
            // Call the next delegate/middleware in the pipeline	 
            return next(context);	 
        }	 
    }	 
}
```

Easy peasy! Now, if you rerun the application, we'll see the logs populated with the properties defined in the Correlation-Context header.

## Summary
In this article, we've seen how from ASP .NET Core 3.0 correlating traces between our (micro)services, it's easier than ever. Hope it helps!