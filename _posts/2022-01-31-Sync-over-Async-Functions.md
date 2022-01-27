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

***NOTE: The underpining platform for Durable Functions is implemented using [Durable Task Framework](https://github.com/Azure/durabletask) library which manages and externalizes state by persisting to Azure Storage, Azure Service Fabric and few other options.*** 

We will see hone one of the key element [Task Hub](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-task-hubs?tabs=csharp) is highly leveraged for coordinating orchestrations and tasks and   [here](https://github.com/Azure/durabletask/wiki/Core-Concepts) talk about how messages are reliably passed between orchestrations and workers


Ok so now with this lets look at how the implementation looks. I have uploaded the simplified code [here](https://github.com/AdiThakker/SyncOverAsync_Functions) 

![image]({{site.url}}/images/sync-async-df.png)

***NOTE: This sample uses .NET 6 and Functions ver. 4, so you will not see Program.cs file and it also leverages some of the new features like globalusings, OpenAPI support etc.***

So the class that implments this functionality is ***SyncOverASyncApi.cs*** and the key snipptes are shown below:

~~~csharp

    private const string OrchestrationFunctionName = "Sync-Over-Async-Api-DurableFunction";
    const string OrchestrationComplete = "weather_async_response";

    public SyncOverAsyncApi(ILogger<SyncOverAsyncApi> logger) => Logger = logger;

    [FunctionName("SyncOverAsyncApi_WeatherRequest")]
    [OpenApiOperation(operationId: "GetWeatherAsync", tags: new[] { "weather" })]
    [OpenApiResponseWithBody(statusCode: HttpStatusCode.OK, contentType: "text/plain", bodyType: typeof(string), Description = "The OK response")]
    public async Task<HttpResponseMessage> GetWeatherAsync([HttpTrigger(AuthorizationLevel.Anonymous, "get")] HttpRequestMessage req,
        [DurableClient] IDurableOrchestrationClient starter)
    {
        // Generate a random request Id
        var requestId = randomGenerator.Next(Int32.MaxValue).ToString();

        // Start new orchestration
        await starter.StartNewAsync(OrchestrationFunctionName, requestId.ToString());
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

    [FunctionName(OrchestrationFunctionName)]
    public async Task<string> RunOrchestrator([OrchestrationTrigger] IDurableOrchestrationContext context) => await context.WaitForExternalEvent<string>(OrchestrationComplete);

    [FunctionName("SyncOverAsyncApi_WeatherReply")]
    public async Task WeatherReply([BlobTrigger("weather-results/{name}", Connection = "blobConnection")] Stream myBlob, string name,
                                   [DurableClient] IDurableOrchestrationClient client)
    {
        var requestId = name.Remove(name.IndexOf('.'));
        await client.RaiseEventAsync(requestId, OrchestrationComplete, new StreamReader(myBlob).ReadToEnd());
        this.Logger.LogInformation($"Received reply for Request:{name}");
    }
~~~

In the above snippet, you can see that once a request is received, the DurableClient in the ***GetWeatherASync*** method starts a new Orchestration by calling ***StartNewAsync*** passing in the Orchestration Function Name and the request id. It then waits for that orchestration to complete or timeout (60 seconds) to occur by awaiting ***WaitForCompletionOrCreateCheckStatusResponseAsync***

The ***RunOrchestrator*** our Orchestrator Function gets invoked, it uses the OrchestrationTrigger's context's ***WaitForExternalEvent*** method by passing in the event name.

The ***WeatherReply*** function is our External Event subscriber(i have used Blob trigger instead of Service Bus Topic) since i was testing it locally but the idea remains the same.

Once that function is invoked, it calls the DurableClient's ***RaiseEventAsync*** to complete it.


Following is the snapshot of how the state is maintained in my local storage emulator

- TaskHubName


-- Container


-- Status Table

So there you see folks, implementing this pattern was really easy using Durable Functions. As you can see with the stateful patterns that are supported, we just scratched the surface.


More to come!!!!
