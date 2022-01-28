---
layout:     post
title:      Sync over Async Web API - Durable Functions
date:       2022-01-28
summary:    This post builds on previous post and explores how to implement same logic in serverless world using Durable Functions.
categories: .NET, Web Api, Asynchronous, Durable Functions 
---

In the [previous]({{site.url}}/Sync-over-Async-WebApi) post, we explored how you could control completion of a synchronous web request over an asynchronous event by using [Task Completion Source](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcompletionsource?view=net-6.0). 

For the project that I was working on, we were using serverless architecture by leveraging [Azure Functions](https://azure.microsoft.com/en-us/services/functions/#features) and as a principle adhered to stateless implementations and avoided long running stateful patterns. But with this use case, we now required to maintain some stateful logic i.e. block the incoming request and only unblock it when an external event occured (in our case it was receiving a reply on a service bus topic). 

Now... we could have implemented this using the Task Completion Source, we saw earlier but that would mean we will have to write additional code, esp. for hardening and persisting state management outside of the function. All of which is fine.... but wait, functions already solves this for us 😊 by using extension called [Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=csharp). 

When I first started reading about Durable Functions, I was intrigued by its API and the type of stateful application patterns that could be implemented using this sdk. 

***NOTE: I would highly encourage you to read about its application patterns [here](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=csharp#application-patterns) as the framework really simplifies some complex patterns which require state management, like a [Saga](https://microservices.io/patterns/data/saga.html) or [Actor](https://en.wikipedia.org/wiki/Actor_model) model...more on that later in a future blog post.***

OK, back to our use case...while ours looks similar to the [Async HTTP API](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=csharp#async-http) pattern, its not exactly the same, as we are not using polling here and instead we need a callback via external event as described previously. 

One of the use cases described in the docs is, [Handling External Events](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-external-events?tabs=csharp) and as  the name indicates, that was exactly what we were looking for. So in order to implement this pattern, we needed:

- An [HTTP-Triggered Client function](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview#client-functions): Our HTTP-triggered  non-orchestrator function, mainly for receiving HTTP request and starting the Orchestrator.

- An [Orchestrator function](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview#orchestrator-functions): As the name indicates, this function starts and waits for that external event notification to resume / unblock in-flight request.  

- A [Topic-Triggered Client function](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview#client-functions): And finally, our Topic-triggered non-orchestrator function which subscribes to the topic and raises the event for the Orchestrator to proceed.

***NOTE: If you know what I am talking about, you are good...otherwise please read durable function types [here](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview) first before reading further.***

OK so equipped with this, lets look at how the implementation looks. 

I have uploaded the simplified code [here](https://github.com/AdiThakker/SyncOverAsync_Functions) and have also made one modification, ***instead of subscribing to a service bus topic, I have used a blob trigger so I can test this locally without provisioning any resources*** since the overall idea of external event invocation, still stays the same.

Following is the sequence diagram:   

![image]({{site.url}}/images/sync-async-df.png)

***NOTE: This sample uses .NET 6 / C# 10 and Functions (ver. 4), so refer to the sources for [globalusings](https://devblogs.microsoft.com/dotnet/welcome-to-csharp-10/), [OpenAPI](https://github.com/azure/azure-functions-openapi-extension) features.***

If you look at the source code, the interesting snippets are in the [SyncOverAsyncApi.cs](https://github.com/AdiThakker/SyncOverAsync_Functions/blob/main/SyncOverAsync_Functions/SyncOverAsyncApi.cs) class:

~~~csharp
[FunctionName(nameof(GetWeatherAsync))]
[OpenApiOperation(operationId: "GetWeatherAsync", tags: new[] { "weather" })]
[OpenApiResponseWithBody(statusCode: HttpStatusCode.OK, contentType: "text/plain", bodyType: typeof(string), Description = "The OK response")]
public async Task<HttpResponseMessage> GetWeatherAsync([HttpTrigger(AuthorizationLevel.Anonymous, "get")] HttpRequestMessage req,
                                                        [DurableClient] IDurableOrchestrationClient starter)
{
    // Generate a random request Id
    var requestId = randomGenerator.Next(Int32.MaxValue).ToString();

    // Start new orchestration and pass requestId as instance id.
    await starter.StartNewAsync(nameof(RunOrchestrator), requestId);
    this.Logger.LogInformation($"Started orchestration for {requestId}");

    // Wait for orchestration to complete or timeout to occur
    var completion = await starter.WaitForCompletionOrCreateCheckStatusResponseAsync(req, requestId.ToString(), TimeSpan.FromSeconds(60));
    if (completion.StatusCode != HttpStatusCode.OK)
    {
        await starter.TerminateAsync(requestId, "Timeout Occured"); // Log additional context (if any)
        return new HttpResponseMessage(HttpStatusCode.InternalServerError);
    }

    return completion;
}

[FunctionName(nameof(RunOrchestrator))]
public async Task<string> RunOrchestrator([OrchestrationTrigger] IDurableOrchestrationContext context) => await context.WaitForExternalEvent<string>(eventName);

[FunctionName(nameof(WeatherResponse))]
public async Task WeatherResponse([BlobTrigger("weather-results/{name}", Connection = "blobConnection")] Stream myBlob, string name,
                                [DurableClient] IDurableOrchestrationClient client)
{
    // Retrieve the requestid (this one looks at the file name)
    var requestId = name.Remove(name.IndexOf('.'));
    
    // Send notification to the orchestration instance specifying the event completion
    await client.RaiseEventAsync(requestId, eventName, new StreamReader(myBlob).ReadToEnd());
    this.Logger.LogInformation($"Received reply for Request:{name}");
}
~~~

In the above snippet, you can see that once a request is received, the DurableClient in the ***GetWeatherAsync method***, starts a new Orchestration by calling *StartNewAsync*, passing in the Orchestration function name and the requestId as the instance. It then waits for that orchestration to complete or timeout by awaiting *WaitForCompletionOrCreateCheckStatusResponseAsync*

The ***RunOrchestrator method***, which is our Orchestrator function gets invoked and it uses the passed in context's *WaitForExternalEvent* method to wait by passing in the event.

The ***WeatherResponse method*** is our external Event subscriber for Blob trigger. Once that function is invoked, it calls the DurableClient's ***RaiseEventAsync*** to complete the event.

The glue to all of the above code is [Orchestration Client](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-bindings?tabs=csharp#orchestration-client) and [Orchestration Context](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-bindings?tabs=csharp#orchestration-trigger)

***NOTE: The underpining platform for Durable Functions is implemented using [Durable Task Framework](https://github.com/Azure/durabletask) library which manages, externalizes state by persisting to Azure Storage, Azure Service Fabric, etc. The construct of [Task Hub](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-task-hubs?tabs=csharp) is highly used for storing orchestration instances and history. You can read further about several of these [here](https://github.com/Azure/durabletask/wiki/Core-Concepts).*** 

What I have shown below is an output of my local storage Task hub when running this code, you can see ***TestHubNameInstances*** table maintains the request status: 

![image]({{site.url}}/images/sync-async-df-1.png)

Where as the ***TestHubNameHistory*** table maintains the complete history of all the requests:

![image]({{site.url}}/images/sync-async-df-2.png)

So there you see folks, we have just scratched the surface of Durable functions and it is very powerful. 

BTW, I am also very excited to read about their availability in the isolated process model in its [roadmap](https://techcommunity.microsoft.com/t5/apps-on-azure-blog/net-on-azure-functions-roadmap/ba-p/2197916), so more to explore!!!!
