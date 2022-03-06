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


- [RulesBuilder]() This class, as the name indicates is mainly responsible for building the Rules engine runtime by looking at the [RulesConfiguration]() and  

All ths source code is avaialable [here]() and it leverages the custom template that we discussed in the [previous post]()


Let me know what your thoughts are!!! 









