---
layout:     post
title:      Implementing Resiliency in .NET 
date:       2024-04-10
summary:    A simple reference / concept for how to custom implement Resiliency in .NET.
categories: .NET, Functional Programming
---

Over the last weekend, I was trying to explain resiliency as a concept to a team of developers and how it can be implemented in .NET. This post is the outcome of that discussion.

In the world of software development, resiliency is key to maintaining robust and reliable applications, especially in distributed environments where failures are not just possible—they are inevitable. A resilient application can gracefully handle and recover from failures, ensuring minimal disruption to the user experience. The two concepts on which this builds are [Retries](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry) and [Circuit Breaker](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker) patterns.

**NOTE**: I'll not explain the above concepts as the links provided are more than enough to understand them. However just to give a brief overview:

- **Retries** is a fundamental strategy for handling transient failures (like temporary network issues). By automatically retrying a failed request, applications can often overcome a failure that is only temporary in nature, but this can also lead to cascading failures if not implemented correctly, Hence, the Circuit Breaker pattern.
  
- **Circuit Breaker** pattern helps control an interaction with an external system by stopping cascading failures and giving the failing system time to recover. It also helps to avoid the situation where an application is continuously trying to execute an operation that will always fail, allowing it to gracefully degrade functionality when needed.

so with that said, let's dive into how you can enhance the resiliency of your .NET applications using retries and the Circuit Breaker pattern.

All of the code is uploaded [here](https://github.com/AdiThakker/projects/tree/main/ResiliencySample)

**NOTE**: This code was built in a short time and is just to explain the concepts. For production use, you should use powerful libraries like [Polly](https://www.pollydocs.org/)

## Implementation

All of the code resides in this [Resiliency](https://github.com/AdiThakker/projects/blob/main/ResiliencySample/Resiliency.cs) class, which has following functions: `Try<T>` and `Break<T>` as shown below:

~~~csharp
 public static T Try<T>(string correlationId, Func<T> run, int currentRetry = 0)
{
    try
    {
        Console.Write($"Attempt: {currentRetry} - ");
        var currentState = circuitStatus.TryGetValue(correlationId, out CircuitState value);
        return value switch
        {
            CircuitState.Closed => run(),
            CircuitState.Open => Break(correlationId, run, "Circuit is Open cannot perform action, Escalating to Circuit-Break!!!"),
            CircuitState.HalfOpen => run(), // go ahead and execute. you could probably use a different strategy here
            _ => throw new NotImplementedException()
        };
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        var result = Retry(currentRetry);
        return result switch
        {
            (true, _) => Try(correlationId, run, result.current),
            (false, _) => Break(correlationId, run, "Retries Exhausted, Escalating to Circuit-Break!!!")
        };
    }

    static (bool retry, int current) Retry(int currentRetry)
    {
        // Check the exception Type and do retries for specific amount of time
        if (currentRetry >= 3) return (false, -1);

        currentRetry++;
        Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, currentRetry))).GetAwaiter().GetResult();
        return (true, currentRetry);
    }
}
~~~

The `Try<T>` method is the main method that you would call to perform the action. It takes a correlationId, a function to run, and the currentRetry count. It first checks the circuit status and then runs the function. If the function fails, it retries the function based on the currentRetry count. If the retries are exhausted, it escalates to the `Break` method.

~~~csharp
static T Break<T>(string correlationId, Func<T> run, string message)
{
    Console.WriteLine($"{message}");
    circuitStatus.AddOrUpdate(correlationId, CircuitState.Open, (key, state) => state = CircuitState.Open);

    // circuit-break
    var latestState = Enumerable.Range(1, 3)
                        .Select(count =>
                        {
                            Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, count))).GetAwaiter().GetResult();  // you can probably sleep till threshold reached                              
                            var status = Number.Next(int.MaxValue) % 2 == 0; // backup operation status
                            Console.WriteLine($"Perform backup operation {count}: status succeeded - {status}");
                            return status;
                        })
                        .Count(_ => _ == false)
                        .GetCurrentState();

    Console.WriteLine($"Timeout complete, Latest State : {latestState}.");
    circuitStatus.AddOrUpdate(correlationId, latestState, (key, state) => state = latestState);
    
    // resume
    return Try(correlationId, run);
}

private static CircuitState GetCurrentState(this int failures )
{
    return failures switch
    {
        <= 0 => CircuitState.Closed,
        1 or 2 => CircuitState.HalfOpen,
        3 => CircuitState.Open,
        > 3 => throw new Exception("No possible")
    };
}
~~~

The `Break<T>` method is called when the retries are exhausted. It sets the circuit status to open and then performs a backup operation. If the backup operation suceeds (the above code leaves that out from the implementation), it sets the circuit status to open; otherwise, it sets the circuit status to half-open. It then resumes the original operation.

## Sample Output

One of the runs of the code is shown below:

![Setup]({{site.url}}/images/run-result-1.png){:height="500px" width="700px"}


## Considerations

- The above code is just a simple implementation of the concepts. In production, you should use libraries like [Polly](https://www.pollydocs.org/) which are more powerful and have more features.
  
- The current implementation uses an in-memory dictionary to manage states. For distributed applications, consider using a distributed cache (like Redis) or a persistent store to maintain the state across multiple instances and ensure consistency.

- The retry logic ideally should be sensitive to the type of exception. Not all exceptions warrant a retry.
  
-  Remember that the Circuit Breaker pattern is not a silver bullet. It is just one of the many tools in your toolbox to build resilient applications. You should also consider other patterns like [Bulkhead](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead) to build a comprehensive resiliency strategy.
  
- In the Half-Open state, the code should allow a limited number of requests to test if the underlying problem has been resolved. Also, the transition from Open to Half-Open is not implemented in the above example and is left as an exercise for the reader, this could be passed as a different strategy to the `Try` method for e.g. something that automates the transition based on the time elapsed, perhaps as a background task or timer.


