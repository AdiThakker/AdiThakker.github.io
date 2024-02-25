---
layout:     post
title:      Embracing Railway-Oriented Programming in .NET – A Practical Approach 
date:       2024-02-15
summary:    A Reference for how to leverage Railway Oriented Programming concepts in .NET.
categories: .NET, Functional Programming
---

I am huge fan of F# and functional programming in general. I have previously used F# mainly for learning purposes and for small projects. I have always been fascinated by the concept of Railway Oriented Programming (ROP) and how it can be used to handle errors in a functional way.

In this post we will see how we can leverage those concepts in .NET. mainly using C#.

## What is Railway Oriented Programming?
ROP, a functional programming concept, visualizes error handling like two parallel train tracks: one for success and one for failure. It simplifies the process of handling errors, making your code more readable and maintainable.

So, instead of using exceptions, we can use ROP to handle errors in a more functional way. This is done by chaining functions together, where each function returns a result that can be either a success or a failure. This result is then passed to the next function in the chain, which can then decide what to do based on the result.

## Our Reference example

All the source code is uploaded [here](https://github.com/AdiThakker/ROP) 

### Result Type

We start by defining a result interface and class, IResult<TSuccess, TFailure> and Result<TSuccess, TFailure>. This serves as the foundation, encapsulating either a success value or a failure value as shown below.

~~~csharp
public interface IResult<TSuccess, TFailure>
{
}

public class Result<TSuccess, TFailure> : IResult<TSuccess, TFailure>
{
    public TSuccess Success { get; set; }

    public TFailure Failure { get; set; }

    public Result(TSuccess success)
    {
        Success = success;
        Failure = default;
    }

    public Result(TFailure failure)
    {
        Failure = failure;
        Success = default;
    }

    public void Deconstruct(out TSuccess? success, out TFailure? failure) { success = Success; failure = Failure; }
}
~~~

Imagine your function as two parallel tracks. The success track continues when operations are successful. The failure track is taken when an error occurs. With Railway Oriented Programming, these tracks never intersect, simplifying error handling

### Bind function

Think of Bind as a switch on the railway. If the incoming train (input) is on the success track, Bind applies a function to transform the value. If it's on the failure track, Bind passes the failure along without applying the function. This method elegantly handles the switch between success and failure paths, which is shown below:

~~~csharp
public static Func<Result<TValue, TFailure>, Result<TSuccess, TFailure>> Bind<TValue, TSuccess, TFailure>(Func<TValue, Result<TSuccess, TFailure>> map)
{
    return input =>
        {
            var (success, failure) = input;
            return (success, failure) switch
            {
                (_, null) => map(success),
                (null, _) => new Result<TSuccess,TFailure>(failure),
                _ => throw new NotImplementedException(),
            };
        };
}
~~~

Applying the Bind function via an extension method, we can chain functions together, as shown below:

~~~csharp
public static Result<TSuccess, TFailure> Then<TValue, TSuccess, TFailure>(this Result<TValue, TFailure> instance, Func<TValue, Result<TSuccess, TFailure>> map) => Bind(map)(instance);
~~~

### Validation Example

Shown below is our validation example using the above constructs.

~~~csharp
public class Customer
{
    public string? Name { get; set; }

    public int Age { get; set; }
}

public class CustomerResult : Result<Customer, Exception>
{
    public CustomerResult(Customer success) : base(success) { }

    public CustomerResult(Exception error) : base(error) { }
}

public static class CustomerValidation
{
    public static CustomerResult ValidateName(Customer customer) => !string.IsNullOrWhiteSpace(customer.Name) ? new CustomerResult(customer) : new CustomerResult(new InvalidDataException("Name cannot be empty"));

    public static CustomerResult ValidateAge(Customer customer) => customer.Age is > 0 and < 100 ? new CustomerResult(customer) : new CustomerResult(new InvalidDataException("Age Invalid"));
}
~~~


So there you have it, we have successfully integrated SignalR with APIM and exposed the endpoints to our clients. I had to keep this one short with minimum examples, but I hope this helps you get started with your SignalR and APIM integration.
