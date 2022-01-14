---
layout:     post
title:      Sync over Async Web API
date:       2021-01-20 12:32:18
summary:    This one explores how to leverage TaskCompletionSource to control the completion of a Web API request.
categories: Blogging, General
---

While working on one my recent projects, we were faced with the requirement where, when a web request came in, it had to be blocked for an unspecified amount of time, till an **external event signaled** for its completion and then it could unblock and return back to its caller. 

Not clear, I think the following diagram explains the exact use case:

![Setup]({{site.url}}/images/sync-over-async-1.png){:height="500px" width="800px"}

In the above diagram, you can see that the flow between steps 2 and 4 is asynchronous where the Web Api has to publish a message to a topic, then wait on a subscribed topic to receive its response back and only then it can unblock that synchronous request.  

The requirement that made this interesting was associating the asynchronous action (*in this case, service bus topic subscription callback*) back to its synchronous operation **without leaving that method's body**.

Now, we are all aware of the [Asynchronous Programming Patterns](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/) and [Task](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task?view=net-6.0) constructs in .NET, so this was going to be at the center of this use case and therefore i needed something which could allow me to act as a producer of a task and at the same time have control over that task's lifetime and that's where our old friend [Task Completion Source](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcompletionsource?view=net-6.0) comes into action.


The key thing with TCS is that there is no scheduled delegate associated with it and therefore you can control the lifetime of the task associated with it by calling its SetResult, SetException and SetCanceled and other methods. 

There is an excellent article by [Stephen Toub] on this [here](https://devblogs.microsoft.com/pfxteam/the-nature-of-taskcompletionsourcetresult/) which you can refer to for more details.


So with this lets look at how its implemented.

NOTE: I have simplified the [code](https://github.com/AdiThakker/SyncOverAsync) to demonstrate the pattern. 

Following are the interesting elements

- Initial scaffolding of the [minimal Web Api](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis?view=aspnetcore-6.0) 

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

In the above code snippet the *MapGet* routes are where the synchrnous and asynchronous methods are called.

The synchrnous *GetWeather* is straight from the built-in template

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

The following is asynchronous *GetWeatherAsync* method

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

as you can see, *GetWeatherAsync* method accepts a correlationId which is used as a key to track the TCS that is associated with that request. 

Once the api sync request comes in, it kicks off the *GetWeatherAsyncCompletion* passing in that correlationId. The *GetWeatherAsyncCompletion* is where the completion / exception of that TCS is signaled, thereby unblocking the task assoicated with that TCS of that correlation/request. 


So there you see folks TCS really simplified our Sync over Async use case in the Web Api scenario. I am sure there are other ways to implement this pattern. Feel free to share your thoughts or leave comments, if any.

BTW, in the next post we will see how we can approach this in **serverless** world using **Azure Functions**, which is what we ended up implementing in actual use case.

so...stay tuned!!!



