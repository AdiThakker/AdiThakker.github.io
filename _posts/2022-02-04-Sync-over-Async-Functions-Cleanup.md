---
layout:     post
title:      Sync over Async Web API - Durable Functions (Clean up)
date:       2022-02-04
summary:    Addendum to the previous post about clean up of Durable Orchestration History.
categories: .NET, Web Api, Asynchronous, Durable Functions 
---

I forgot to mention about clean up of durable function orchestration history in my [previous]({{site.url}}/Sync-over-Async-Functions) post, so I thought i'll include that here.

As we saw in the last post, the entire Orchestration history is maintained in our ***TestHubNameHistory*** table instance and over time these records will keep growing, so a good practice would be to clean those up on periodic basis. 

The [DurableClient](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.webjobs.extensions.durabletask.idurableclient?view=azure-dotnet) provides a helpful method for that, called [PurgeInstanceHistoryAsync](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.webjobs.extensions.durabletask.idurableorchestrationclient.purgeinstancehistoryasync?view=azure-dotnet#microsoft-azure-webjobs-extensions-durabletask-idurableorchestrationclient-purgeinstancehistoryasync(system-datetime-system-nullable((system-datetime))-system-collections-generic-ienumerable((durabletask-core-orchestrationstatus)))). This method provides couple overloads and as you will see, I have used one of them in a [TimerTrigger](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer?tabs=csharp) function for performing cleanup in the [SyncOverAsyncApi.cs](https://github.com/AdiThakker/SyncOverAsync_Functions/blob/main/SyncOverAsync_Functions/SyncOverAsyncApi.cs) class, shown below:

~~~csharp
[FunctionName(nameof(CleanUpOldWeatherResponsesAsync))]
public async Task CleanUpOldWeatherResponsesAsync([TimerTrigger("0 0 0 * * *")] /* execute every day */ TimerInfo myTimer, [DurableClient] IDurableOrchestrationClient client)
{
    var result = await client.PurgeInstanceHistoryAsync(DateTime.UtcNow.AddDays(-15), DateTime.UtcNow.AddDays(-1), new List<OrchestrationStatus> { OrchestrationStatus.Completed, OrchestrationStatus.Terminated });
    this.Logger.LogInformation("Cleaned up records: ", result?.InstancesDeleted);
}
~~~

The above method ***CleanUpOldWeatherResponsesAsync*** is straightforward, it executes daily and accepts DurableClient as one of the parameter provided by the runtime. The overload of ***PurgeInstanceHistoryAsync***, takes the date range and the orchestration statuses to be purged. 

***NOTE: If you test this locally, you might run into [BadRequest](https://github.com/Azure/azure-functions-durable-extension/issues/2007) issue depending on your setup***.

Finally, there are some performance considerations to take into account when using these APIs, if you need an alternative / faster way to clean up your taskhub state, refer to [this](https://dev.to/cgillum/resetting-your-durable-task-hubs-azure-storage-state-2ome) article by Chris Gillum.





