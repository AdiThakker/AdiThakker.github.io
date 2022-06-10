---
layout:     post
title:      Event Aggregation Workflow Via Durable Functions
date:       2022-06-10
summary:    This post explores one way of implementing Event Aggregation Workflow via Durable Functions.
categories: Azure Durable Functions, .NET
---

Recently at work... we came across a workflow, where we had to aggregate certain type of events by leveraging [Durable Functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=csharp). 

The use case was something like the this: 

***When a certain primary event is received by a workflow step, that step has to wait for all of its dependencies to arrive before proceeding further with the next one. However, if a certain threshold timeout occurs, then that step can skip waiting for its dependencies.***

I think the following sequence diagram would help explain this further:

![image]({{site.url}}/images/Durable-Functions-2.png)

In the above diagram, when ***Event X (primary event)*** is received on a topic, the Event Aggregator step starts a ***threshold timer (24 hours)*** in this case and then simultaneously waits for all of its dependencies to arrive (on different topics). 

If all of the dependencies arrive within that threshold, the Event Aggregator moves on and publishes its ***Completed*** status to the next topic. 

However if a timeout occurs, then the Event Aggregator abandons the event aggregation and publishes its ***Incomplete*** status.

I wanted to test this out using a small PoC and have uploaded that code [here](https://github.com/AdiThakker/Azure.DurableFunctions.EventAggregator)

***NOTE: The following code has some similarities to one of my [previous]({{site.url}}/Sync-over-Async-Functions) post. You can read more details there or go to official [docs](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=csharp)***

Now, lets dive into some of the interesting parts of [EventAggregatorOrchestrator](https://github.com/AdiThakker/Azure.DurableFunctions.EventAggregator/blob/main/Azure.DurableFunctions.EventAggregator/DurableFunction/EventAggregatorOrchestrator.cs) class:

- First, defining the events and its dependencies in a Dictionary is shown below:

~~~csharp
private static readonly ConcurrentDictionary<string, List<string>> dependencies;

static EventAggregatorOrchestrator()
{
	dependencies = new ConcurrentDictionary<string, List<string>>();
	dependencies.TryAdd("Event A", new List<string>() { "A-1", "A-2" });
	dependencies.TryAdd("Event B", new List<string>() { "B-1" });
	dependencies.TryAdd("Event C", new List<string>() { "" });
}

~~~

- Next, our [Durable Client functions](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview#client-functions) which are the entry points for receiving all (primary & dependencies) events via EventGrid. 

~~~csharp

[FunctionName("Event-Subscriber")]
public async Task ReceiveEventsAsync([EventGridTrigger] EventGridEvent eventGridEvent, [DurableClient] IDurableClient client)
{
	Logger.LogInformation($"Received Event: {eventGridEvent.Data.ToString()}");
	await ReceiveOrUpdateEventsAsync(eventGridEvent, client);
}

[FunctionName("Dependencies-Subscriber")]
public async Task ReceiveDependenciesAsync([EventGridTrigger] EventGridEvent eventGridEvent, [DurableClient] IDurableClient client)
{
	Logger.LogInformation($"Received Dependency: {eventGridEvent.Data.ToString()}");
	await ReceiveOrUpdateEventsAsync(eventGridEvent, client);
}

~~~

The above functions end up calling a helper method ***ReceiveOrUpdateEventsAsync*** which has the logic for managing the [Orchestrator function](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview#orchestrator-functions). This logic either starts a new orchestration, or raises an [external event](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-external-events?tabs=csharp) to resume a ***running*** orchestrator. The below snippet shows that logic:

~~~csharp

private async Task ReceiveOrUpdateEventsAsync(EventGridEvent eventGridEvent, IDurableClient client)
{
	// Get orchestration Status
	var orchestration = await client.GetStatusAsync(eventGridEvent.Subject); // Subject is Unique for Testing
	if (orchestration == null)
	{
		var instance = await client.StartNewAsync(@"Event-Aggregator-Orchestrator", eventGridEvent.Subject, eventGridEvent);
		Logger.LogInformation($"Started new Orchestration instance {instance} for {orchestration}");
	}
	else
	{
		_ = orchestration.RuntimeStatus switch
		{
			OrchestrationRuntimeStatus.Running => RaiseEventForOrchestration(),
			_ => StartOrchestration()
		};
	}

	async Task RaiseEventForOrchestration()
	{
		await client.RaiseEventAsync(orchestration.InstanceId, @"Event-Aggregator-Orchestrator", eventGridEvent);
		Logger.LogInformation($"Received dependency event for {orchestration}");
	}

	async Task StartOrchestration()
	{
		var instance = await client.StartNewAsync(@"Event-Aggregator-Orchestrator", eventGridEvent.Subject, eventGridEvent);
		Logger.LogInformation($"Started new Orchestration instance {instance} for {orchestration}");
	}
}
~~~

- Finally, our [Orchestrator Function](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview#orchestrator-functions) which manages the workflow of event aggregation and/or timeout.

~~~csharp

[FunctionName("Event-Aggregator-Orchestrator")]
public async Task AggregateEventsAsync([OrchestrationTrigger] IDurableOrchestrationContext context)
{
	try
	{
		// Get the list of dependencies to aggregate for a given event
		var receivedEvent = context.GetInput<EventGridEvent>();
		string status = "";

		// Check if any dependencies     
		if (dependencies.TryGetValue(receivedEvent.Subject.ToString(), out List<string> dependenciesList))
		{
			if (dependenciesList.Any())
			{
				using var cts = new CancellationTokenSource();
				
				// Create timer to wait for dependencies to be received
				var endTime = context.CurrentUtcDateTime.Add(TimeSpan.FromSeconds(30)); // Durable Timer                         
				var timeoutTask = context.CreateTimer<List<string>>(endTime, default, cts.Token);

				// Start tracking dependencies
				var dependenciesRemaining = dependenciesList.ToList();
				while (dependenciesRemaining.Any())
				{
					// wait for dependencies to arrive
					var dependenciesTask = context.WaitForExternalEvent<EventGridEvent>(@"Event-Aggregator-Orchestrator");
					var completedTask = await Task.WhenAny(timeoutTask, dependenciesTask);
					if (completedTask == dependenciesTask)
					{
						if (dependenciesTask.Result != null)
						{
							dependenciesRemaining.Remove(dependenciesTask.Result.EventType);
							if (dependenciesRemaining.Count == 0)
							{
								// All dependencies received
								status = "All dependencies received!";
								cts.Cancel();
								break;
							}
						}
					}
					else if (completedTask == timeoutTask)
					{
						// Timeout
						status = $"Timeout Occured, dependencies not received: {dependenciesList.Count}";
						break;
					}
				}
			}
			else
			{
				status = "No dependencies, so moving on!";
			}
		}
		//
		await context.CallActivityAsync(@"Event-Status-Publisher", status);
	}
	catch (TaskCanceledException ex)
	{
		if (!context.IsReplaying)
			Logger.LogError(ex.ToString());
	}
	catch (Exception ex)
	{
		if (!context.IsReplaying)
			Logger.LogError(ex.ToString());
	}
}
~~~

Some of the key points to note in the above snippet are creation of [Durable Timer](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-timers?tabs=csharp) via ***context.CreateTimer***. 

Next is waiting for the dependencies collection via [external event](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-external-events?tabs=csharp) using ***context.WaitForExternalEvent***. 

Also, since the orchestration history is replayed I had to be careful of not introducing any non-deterministic logic. Read more about that [here](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-orchestrations?tabs=csharp#reliability).

Finally, once the Event Aggregation finishes, it ends up publishing its status via an [Activity Function](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview#activity-functions) which is shown below:

~~~csharp

[FunctionName("Event-Status-Publisher")]
public void PublishStatus([ActivityTrigger] string status)
{
	Logger.LogInformation(status);
}

~~~
Running the workflow for both use cases (timeout and all dependencies received) is shown below:

![image]({{site.url}}/images/Durable-Functions-3.png)

![image]({{site.url}}/images/Durable-Functions-4.png)

![image]({{site.url}}/images/Durable-Functions-5.png)

![image]({{site.url}}/images/Durable-Functions-6.png)

So, there you see folks! Implementing this durable workflow was very convenient, It takes a little used to adhering to [Orchestration code constraints](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-code-constraints) but I guess with more practice, you get used to it. 

In the next post, I'll try to take this further and see if I can implement this using [Durable Entities](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-dotnet-entities) to compare and contrast the implementation.