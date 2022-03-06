---
layout:     post
title:      Simple Rules engine via Expression Trees
date:       2022-03-06
summary:    This post explores a simple rules engine implementation leveraging Expression Trees in .NET.
categories: .NET, ExpressionTrees, Meta Programming 
---

In this post, we will explore how we leveraged [Expression Trees](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/expression-trees/) and [Configuration](https://docs.microsoft.com/en-us/dotnet/core/extensions/configuration) to create / execute simple rules engine component.

We had a requirement, where our upstream components / services, could forward custom data points such as errors, custom dimensions, etc. to our rules / alarms component.... and based on configured criteria, we should be able to execute custom logic such as forwarding or escalating rules (more on that later). 

An interesting point about this setup was that the rules execution logic was simple but it had to be dynamically invoked based on certain configurable criteria since this could change in the upstream components and also it  should be easily deployable without modifying the source code.


Now, .NET provides us various metaprogramming options (Reflection, Code DOM, etc.) but for our use case, we needed something that was light weight and performant, hence we decided to go forward with Expression Trees. We also leveraged .NET configuration JSON structure to define our mini DSL for our rules execution logic. 

An [example]() of which is shown below:

***NOTE: Our use case for rules configuration was simple...depending on your use case, you can end up defining it accordingly so care must be taken to addd validation in place.The above configuration could really get complex so its important to keep a balance....***


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

The key elements in the above configuration are:

- Criteria: This element is our mini DSL language which follows a convention of an object's "Property1-Property2-Property3 <expression> Value1-Value2-Value3" format. The source generation logic, looks at this text and parses it to dynamically generate the C# code (a function in this case). 

***NOTE: This is just an example of a simple implementation, this can be implemented in any structure you want which then affects your parsing logic.***

The object that defines these properties shown here[](). 

***NOTE: There are some first class properties and also a dictionary (mainly used for extensibility) so that the upstream components could easily send additional Key / values to be considered for rules execution logic. 

-- Rules: This array element defines all the rules that should be executed when that specific criteria is met.  



The below class diagram shows what are the different classes involved in constructing this component.

![image]({{site.url}}/images/classes-et-1.png)


- [RulesEngineService](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Adi.FunctionApp.RulesEngine.Service/RulesEngineService.cs) This is the entry function code that interacts with the Rules logic code. This logic is part of the function app which registers all the dependencies in the [Startup](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Adi.FunctionApp.RulesEngine.Service/Startup.cs) as shown below:

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

- [RulesConfiguration](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Shared/Adi.FunctionApp.RulesEngine.Domain/Models/RulesConfiguration.cs) This is the rules configuration object that's mapped from [appsettings.json](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Adi.FunctionApp.RulesEngine.Service/appsettings.json) 


- [RulesBuilder](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Shared/Adi.FunctionApp.RulesEngine.Domain/Builder/RulesBuilder.cs) This class, as the name indicates is mainly responsible for building the execution runtime by looking at rules configuration

Key methods include:

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

- [RuleExecutor](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Shared/Adi.FunctionApp.RulesEngine.Domain/Executor/RulesExecutor.cs) This class is the actual run time execution of rules. The method that has that logic is shown below:

~~~csharp
public IEnumerable<Task<RuleResult>> Execute(RuleContext input)
{
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

- [ForwardRule](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Shared/Adi.FunctionApp.RulesEngine.Domain/Rules/ForwardRule.cs), [EscalateRule](https://github.com/AdiThakker/Adi.FunctionApp.RulesEngine/blob/main/Source/Shared/Adi.FunctionApp.RulesEngine.Domain/Rules/EscalateRule.cs) are the actual rules that get executed when the criteria matches.

As you can see in the following output, correct rule being executed when criteria is met.

![image]({{site.url}}/images/classes-et-1.png)

All ths source code is avaialable [here]() and it leverages the custom template that we discussed in the [previous post]()


Let me know what your thoughts are!!! 









