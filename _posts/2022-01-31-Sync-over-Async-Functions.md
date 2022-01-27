---
layout:     post
title:      Sync over Async Web API - Durable Functions
date:       2022-01-31
summary:    This post builds on previous post and explores how to implement same logic via Durable Functions.
categories: .NET, Web Api, Asynchronous, Durable Functions 
---

In the [previous]({{site.url}}/Sync-over-Async-WebApi) post we explored how you could control completion of a synchronous web request over an asynchronous event by using [Task Completion Source]((https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcompletionsource?view=net-6.0)) and that was good for understanding the pattern. 

For the project that i was working on, we were using serverless architecture by leveraging [Azure Functions](https://azure.microsoft.com/en-us/services/functions/#features) and as a principle stayed with stateless functions and avoided stateful implementations... but this specific use case now required us to maintain some state (i.e. the incoming request) and only unblock it when an external event (like messaging topic received its reply). (TODO... reword this). 

Now we could have implemented this using the [Task Completion Source] pattern we saw earlier, but wait.... Functions already solves this for us 😊 by using extension called [Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=csharp). 

If you read through the above link, you can see that there are a several stateful application patterns that can be coded using this sdk. While ours looks similar to the [Async HTTP API](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=csharp#async-http) pattern, its not exactly the same, as we are not using polling here and instead we need a callback via external event (actual use case was topic subscription). 

***NOTE: I would highly encourage you to read about the other implementations as the framework really simplifies some complex patterns like a [Saga](https://microservices.io/patterns/data/saga.html) or [Actor](https://en.wikipedia.org/wiki/Actor_model)...more on that later in a future blog post 😊.***


OK, so for our use case we needed an External Event callback / notification and for that, [this] (https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-external-events?tabs=csharp) API looked promising. So in order to implement this logic, we needed:

- [HTTP-Triggered Client function](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview#client-functions): Our HTTP-triggered  non-orchestrator function, mainly for receiving Http request and starting the Orchestrator.

- [Orchestrator function](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview#orchestrator-functions): This function starts and waits for that external event notification to resume / unblock.  

- [Topic-Triggered Client function](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview#client-functions): Our Topic-triggered non-orchestrator function which subscribes to the topic where the reply is expected.

***NOTE: The underpining platform for Durable Functions is implemented using [Durable Task Framework](https://github.com/Azure/durabletask) library which externalizes state by persisting to Azure Storage, Azure Service Fabric and few others. Some of the key concepts such as [Task Hub](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-task-hubs?tabs=csharp),  [here](https://github.com/Azure/durabletask/wiki/Core-Concepts) talk about how messages are reliably passed between orchestrations and workers


Ok so now with this lets look at how the implementation looks....

![image]({{site.url}}/images/sync-async-df.png)

With this we just scratched the surface of Durable Functions....



Confused? I think the following diagram explains the use case:

![image]({{site.url}}/images/sync-async.png)


In the above diagram, you can see the flow between Web Api and Service Bus topic is asynchronous, once the Web Api publishes the message to a service bus topic, it has to wait on a subscribed topic to receive its response back and only then it can unblock that synchronous request.  


The requirement that made this interesting was associating the asynchronous action (*in this case, service bus topic subscription callback*) back to its synchronous operation **without leaving that method's body**.


Now, we are all aware of the [Asynchronous Programming Patterns](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/) and [Task](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task?view=net-6.0) constructs in .NET, so this was going to be at the center of this use case and therefore I needed something which could allow me to act as a producer of a task and at the same time control that task's lifetime and that's where our old friend [Task Completion Source](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcompletionsource?view=net-6.0) comes into action.


The key thing with TCS is that there is no scheduled delegate associated with it and therefore you can control the lifetime of its encapsulated task by calling its SetResult, SetException and SetCanceled and other methods. 


I would encourage you to read [this](https://devblogs.microsoft.com/pfxteam/the-nature-of-taskcompletionsourcetresult/) excellent article by Stephen Toub to find out more about TCS. 


So equiped with this, lets look at how the code looks. **NOTE:** I have simplified the [code](https://github.com/AdiThakker/SyncOverAsync) to demonstrate the logic. This is in no way production ready 😊


- This sample follows the [minimal Web Api](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis?view=aspnetcore-6.0) and the initial scaffolding is straight out of the box. 

~~~csharp

using Microsoft.OpenApi.Models;
using System.Collections.Concurrent;

// Create builder
var builder = WebApplication.CreateBuilder(args);

// register services
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(_ => _.SwaggerDoc("v1", new OpenApiInfo { Title = "Sync Over Async API", Description = "Example showing use of TCS to control task completion", Version = "v1" }));

// build
var app = builder.Build();

// use services
app.UseSwagger();
app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "Sync Over Async API V1"));

// Add Routes
app.MapGet("/weatherforecastsync", () => WeatherForecast.GetWeather());
app.MapGet("/weatherforecastasync", async () => await WeatherForecast.GetWeatherAsync(Random.Shared.Next(int.MaxValue).ToString()));

// Run
app.Run();

~~~

The key snippets in the above code are the *MapGet* routes where the synchronous and asynchronous methods are called for comparison.

The synchronous *GetWeather* is straight from the built-in template, so I won't go into its detail:

~~~csharp
    private static string[] summaries = new[]
    {
        "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
    };

    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);

    public static WeatherForecast[] GetWeather()
    {
       var forecast = Enumerable.Range(1, 5).Select(index =>
       new WeatherForecast
       (
           DateTime.Now.AddDays(index),
           Random.Shared.Next(-20, 55),
           summaries[Random.Shared.Next(summaries.Length)]
       ))
        .ToArray();
        return forecast;
    }
~~~

The following asynchronous *GetWeatherAsync* method is where the action is:

~~~csharp

   static ConcurrentDictionary<string, TaskCompletionSource<WeatherForecast[]>> requests = new ConcurrentDictionary<string, TaskCompletionSource<WeatherForecast[]>>();

    public static async Task<WeatherForecast[]> GetWeatherAsync(string correlationId)
    {
        TaskCompletionSource<WeatherForecast[]> tcs = new TaskCompletionSource<WeatherForecast[]>(correlationId);
        requests.TryAdd(correlationId, tcs);
        await GetWeatherAsyncCompletion(correlationId);
        return await tcs.Task;
    }

    private static async Task GetWeatherAsyncCompletion(string correlationId)
    {
        // simulate background process such as listening on Service Bus topic / external call back, etc.
        // this callback would retrieve the correlationid to signal completion of asynchronous task associated to the 
        // synchronous request
        await Task.Delay(10000);

        // Get the tcs that matches the correlationId
        if (requests.TryGetValue(correlationId, out TaskCompletionSource<WeatherForecast[]> tcs))
            tcs.SetResult(GetWeather());
        else
            tcs.SetException(new Exception("Invalid request"));
    }

~~~

As you can see, the *GetWeatherAsync* method accepts a correlationId which is used as a key to track the TCS that is associated with that request. **NOTE**: We are using a concurrent dictionary *requests* to keep that association.

Once the api request comes in and the association is done, it kicks off the *GetWeatherAsyncCompletion* passing in that correlationId and that is where the completion / exception of that TCS is signaled, thereby unblocking the task assoicated with that TCS of that correlation/request. The *GetWeatherAsync* then just returns that task's result. 

So there you see folks TCS really simplifies our Sync over Async use case in this Web Api scenario. This example can be enhanced to support cancelations and  time outs and I am sure there are other ways to implement this pattern. 

BTW, in the next post we will see how we can approach this in **serverless** world using **Azure Functions**, which is what we ended up implementing in actual use case.

so...stay tuned!!!

Also, feel free to share your thoughts or leave comments, if any.



