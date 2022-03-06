---
layout:     post
title:      Simple Rules engine via Expression Trees
date:       2022-03-06
summary:    This post explores a simple rules engine implementation leveraging Expression Trees in .NET.
categories: .NET, ExpressionTrees, Meta Programming 
---

In this post, we will explore how we leveraged [Expression Trees](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/expression-trees/) and [Configuration](https://docs.microsoft.com/en-us/dotnet/core/extensions/configuration) to create / execute simple rules engine component.

We had a requirement, where our upstream components / services, would forward custom data points such as errors, meta data, etc. to our rules component and based on certain criteria, we would execute configured rules. The interesting this about this setup was that the rules were simple but had to be dynamically configurable, easily deployable without modifying the source code.

Metaprogramming options
.NET provides us various metaprogramming options (Reflection , Code DOM, etc.) We decided to leverage expression Trees
since that fit into our use case here.

We created a simple DSL like JSON configuration that could be used to define our rules execution logic. An example of which is shown below:


***NOTE: The above configuration could really get complex so its important to keep a balance....***


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

- Criteria: This element is our DSL langugage where we defined the ***match*** criteria and based of that we dynamically generate the C# code. This follows a convention of "Property1-Property2-Property3 <expression> Value1-Value2-Value3"

***NOTE: This is just an example of a simple implementation, this can be implemented in any structure you want as this then just becomes your internal parsing logic.***

The contract between the services is a simple class shown here[](). NOTE: There are certain first class properties and for extension points, we included a dictionary so that the upstream components could include secondary data points to be included in the rules execution. 

-- Rules: This array element defines all the rules that should be executed when a sepcific criteria is met and 



The below class diagram shows the different classes  involved in constructing this component.






All ths source code is avaialable [here]() and it leverages the custom template that we discussed in the [previous post]()


Let me know what your thoughts are!!! 









