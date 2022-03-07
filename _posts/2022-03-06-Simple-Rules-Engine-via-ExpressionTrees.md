---
layout:     post
title:      Simple Rules engine via Expression Trees
date:       2022-03-06
summary:    This post explores a simple rules engine implementation using Expression Trees in .NET.
categories: .NET, ExpressionTrees, Meta Programming 
---

In this post, we will explore how we leveraged [Expression Trees](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/expression-trees/) and [.NET Configuration](https://docs.microsoft.com/en-us/dotnet/core/extensions/configuration) to create and execute simple rules engine component.

We had a requirement, where our upstream components or services, would forward custom properties (errors, custom dimensions, etc.) to our rules / alarms component and based on some configured criteria, this component should be able to execute certain rules. 

These rules were static and fairly simple (such as  forwarding / logging, escalation, etc.) but an interesting point about this setup was that it had to be dynamically invoked, since the upstream components could add, change or remove such properties.

We wanted to design this in such a way that changes in the upstream components could easily be honored in the rules component with minimal changes (possibly without source code changes) and therefore making it easily deployable.

Now, .NET provides us various dynamic programming options ([Reflection](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/reflection), [CodeDOM](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/reflection), etc.) however for our use case, we needed something that was light weight and performant, hence we decided to go forward with [Expression Trees](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/expression-trees/). 


***NOTE: Our use case for rules configuration was simple and we could add some validation around it as well. However such setup can get faily complex and therefore should be carefully adopted / evaluated on a case by case basis.***

So lets see how we implemented such requirement. 

We first started with a custom .NET configuration JSON structure to define Configurations list for our rules criteria and execution logic. 

An [example](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Adi.FunctionApp.RulesEngine.Service/appsettings.json) of which is shown below:

~~~JSON
{
  "RulesConfiguration": {
    "Configurations": [
      {
        "Criteria": "Source-ContextType == Order-Product1",
        "Rules": [ "Escalate", "Forward" ]
      },
      {
        "Criteria": "Source == Account",
        "Rules": [ "Forward" ]
      },      
      {
        "Criteria": "Source-Error == Order-QuantityError",
        "Rules": ["Escalate"]
      }
    ]
  }
}
~~~

In the above configuration, you can see that, ***Criteria*** element is our mini DSL language for match and it follows a convention of an object's ***PropertyName condition Value*** structure. This setup can also accomodate ***Property1-Property2 condition Value1-Value2*** for multiple properties match. 

The object that encapsulates these properties is [RuleContext](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Shared/Adi.FunctionApp.RulesEngine.Domain/Models/RuleContext.cs) and our source generation logic (shown later), looks at this and parses it to dynamically generate the C# lambda code (a function in this case). 

***NOTE: If you look at the RuleContext object, there are some first class properties defined, however there is also a dictionary property called Parameters, which is mainly used for extensibility, allowing the upstream components to easily add additional Key - Value pairs to be included in the rules execution logic.*** 

Next is the  ***Rules*** element, which is mainly an array of all the rules that should be executed when that specific criteria is met.

To elaborate the setup further, lets look at the diagram below which shows what are the different classes and their responsibilities.


![image]({{site.url}}/images/classes-et-1.png)


- [Rules Engine Service Function](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Adi.FunctionApp.RulesEngine.Service/RulesEngineService.cs): This function is the client code and is the caller to the Rules engine logic.

~~~csharp
[FunctionName("Dispatcher")]
[OpenApiOperation(operationId: "Dispatcher", tags: new[] { "service" })]
[OpenApiResponseWithBody(statusCode: HttpStatusCode.OK, contentType: "text/plain", bodyType: typeof(string), Description = "The OK response")]
public async Task<IActionResult> Dispatcher(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = "{service}")] HttpRequest req, string service, ILogger log)
{
    log.LogInformation($"Received request for {service}");

    var response = service switch
    {
        "AccountService" => rulesExecutor.Execute(new RuleContext() { Source = "Account" }).Aggregate(new StringBuilder(), (results, result) => results.AppendLine(result.Result.Status)).ToString(),
        "OrderService" => rulesExecutor.Execute(new RuleContext() { Source = "Order", Parameters = new Dictionary<string, string> { { "Error", "QuantityError" } } }).Aggregate(new StringBuilder(), (results, result) => results.AppendLine(result.Result.Status)).ToString(),
        _ => "Invalid request"
    };

    return new OkObjectResult(response);
}
~~~

DI is leveraged in [Startup](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Adi.FunctionApp.RulesEngine.Service/Startup.cs) as shown below:

~~~csharp
public override void Configure(IFunctionsHostBuilder builder)
{
    if (builder is null)
        throw new ArgumentNullException(nameof(builder));

    // Register dependencies
    builder.Services.AddLogging();

    // Register Rules
    builder.Services.AddTransient(typeof(ForwardRule));
    builder.Services.AddTransient(typeof(EscalateRule));

    // Register Builder
    builder.Services.AddOptions<RulesConfiguration>().Configure<IConfiguration>((settings, configuration) => configuration.GetSection(nameof(RulesConfiguration)).Bind(settings));
    builder.Services.AddSingleton<IRulesBuilder<RuleContext, RuleResult>>(sp =>
    {
        var configuration = sp.GetRequiredService<IOptions<RulesConfiguration>>();
        Dictionary<string, IRule<RuleContext, RuleResult>> ruleLookup = new Dictionary<string, IRule<RuleContext, RuleResult>>
        {
            { "Forward", sp.GetRequiredService<ForwardRule>() },
            { "Escalate", sp.GetRequiredService<EscalateRule>() }
        };
        return new RulesBuilder(configuration, ruleLookup);
    });

    // Register Executor
    builder.Services.AddSingleton<IRulesExecutor<RuleContext, RuleResult>>(sp =>
    {
        return new RulesExecutor(sp.GetRequiredService<IRulesBuilder<RuleContext, RuleResult>>(), sp.GetRequiredService<ILogger<RulesExecutor>>());     
    });

}
~~~

All the data points  / properties are wrapped in [Rules Context](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Shared/Adi.FunctionApp.RulesEngine.Domain/Models/RuleContext.cs) object and is pased along downstream.


- [Rules Builder](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Shared/Adi.FunctionApp.RulesEngine.Domain/Builder/RulesBuilder.cs) This class, as the name indicates is mainly responsible for building the execution runtime by looking at [RulesConfiguration](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Shared/Adi.FunctionApp.RulesEngine.Domain/Models/RulesConfiguration.cs) object that's mapped from [appsettings.json](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Adi.FunctionApp.RulesEngine.Service/appsettings.json) 

The main logic for constructing all of this is shown below:

~~~csharp
public IDictionary<Func<RuleContext, bool>, (bool, IEnumerable<IRule<RuleContext, RuleResult>>)> Build()
{
    return this.Configuration.Configurations.Aggregate(new ConcurrentDictionary<Func<RuleContext, bool>, (bool, IEnumerable<IRule<RuleContext, RuleResult>>)>(), (rules, config) =>
    {
        rules.TryAdd(this.GenerateRuleCriteria(config.Criteria),(false, config.Rules.Select(rule => this.Rules[rule])));
        return rules;
    });
}

private Func<RuleContext, bool> GenerateRuleCriteria(string criteria)
{
    var paramExpression = Expression.Parameter(typeof(RuleContext), "context");
    var criteriaBody = criteria.Split(" ", StringSplitOptions.RemoveEmptyEntries);
    (string[] Properties, string[] Values) parseExpressions = (criteriaBody[0].Split("-", StringSplitOptions.RemoveEmptyEntries), criteriaBody[2].Split("-", StringSplitOptions.RemoveEmptyEntries));
    Expression bodyExpression = default;
    for (int i = 0; i < parseExpressions.Properties.Length; i++)
    {
        Expression propertyExpression;
        (Expression Left, Expression Right) expression = BuildPropertyAccessExpression(typeof(RuleContext), paramExpression, parseExpressions.Properties[i], parseExpressions.Values[i]);
        if (criteriaBody[1].Equals("==", StringComparison.InvariantCulture))
            propertyExpression = Expression.Equal(expression.Left, expression.Right);
        else
            propertyExpression = Expression.NotEqual(expression.Left, expression.Right);

        bodyExpression = i == 0 ? propertyExpression : Expression.AndAlso(bodyExpression, propertyExpression);
    }
    
    return Expression.Lambda<Func<RuleContext, bool>>(bodyExpression, paramExpression).Compile();

    (Expression, Expression) BuildPropertyAccessExpression(Type type, ParameterExpression paramExpression, string propertyName, string propertyValue)
    {
        return (type.GetProperty(propertyName) != null)
            ? (Expression.Property(paramExpression, propertyName), Expression.Constant(propertyValue, typeof(string)))
            : (Expression.Property(Expression.Property(paramExpression, "Parameters"), "Item", Expression.Constant(propertyName, typeof(string))), Expression.Constant(propertyValue, typeof(string)));
    }
}
~~~

In the above snippet, you can see that ***GenerateRuleCriteria*** is the main method that parses the configuration to generate the dynamic lambda. The helper ***BuildPropertyAccessExpression*** just builds the property access expression if its a first class property or dictionary. 

- [Rule Executor](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Shared/Adi.FunctionApp.RulesEngine.Domain/Executor/RulesExecutor.cs) This class is the actual run time execution of rules. When ([Rules Context](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Shared/Adi.FunctionApp.RulesEngine.Domain/Models/RuleContext.cs)) is passed along, it gets the lambda match and executes the configured rules as shown below:

~~~csharp
public IEnumerable<Task<RuleResult>> Execute(RuleContext input)
{s
    IEnumerable<IRule<RuleContext,RuleResult>>? GetRulesToExecute(RuleContext input)
    {
        // Get Rules to execute
        var matches = rulesConfiguration
                        .Where(configuration => configuration.Key(input))
                        .Select(criteria => new { Priority = criteria.Value.Item1, Rules = criteria.Value.Item2 });

        if (matches.Any())
            return matches.FirstOrDefault(match => match.Priority == true)?.Rules ?? matches.FirstOrDefault()?.Rules;

        return default;
    };

    var rules = GetRulesToExecute(input);
    if(rules != null && rules.Any())
        return rules.Select(rule => rule.ExecuteAsync(input));

    throw new InvalidOperationException($"No rule to execute for {input.Source} with {input.ContextType}");
}
~~~

The rules that get executed are [Forward Rule](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Shared/Adi.FunctionApp.RulesEngine.Domain/Rules/ForwardRule.cs) or [Escalate Rule](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Shared/Adi.FunctionApp.RulesEngine.Domain/Rules/EscalateRule.cs) based on the configuration.

Following is the output showing those rules execution:

![image]({{site.url}}/images/classes-et-1.png)

I have made all of this avaialable [here](). If you want to explore this further. 

***NOTE: This code also leverages the custom function template that we discussed in one of the [previous post]()

Enjoy!!!














