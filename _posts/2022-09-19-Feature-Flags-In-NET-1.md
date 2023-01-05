---
layout:     post
title:      Feature Flags in .NET - Conditional Feature Flags (Post 1)
date:       2022-09-16
summary:    In this first part of series, we will explore how to implement conditional Feature Flags for executing any custom logic in your .NET code
categories: .NET, Azure, Feature-Flags
---

## Feature Management ##

If you are aware of [Feature Toggles / Feature Management](https://docs.microsoft.com/en-us/azure/azure-app-configuration/concept-feature-management), then you know that it allows you to enable / disable features of your application dynamically. 

Such functionality can be very handy as you can control aspects such as:

- Restricting specific features to only certain set of users.

- Avoiding rollback or immediate hotfix by quickly deactivating a problem feature.

- Stabilize your application by turning off optional features during peak usage.

- Testing in production with only limited users. (sometimes a slippery slope 😉).

In the project that I was involved in recently... there was a requirement, where we had to enable ***Verbose Logging***, after the application was deployed and then automatically disable that after certain configured ***Time-Window*** had passed (say 24 hours). So this led me to explore the Feature Flags path and thereby the [Feature Management API](https://github.com/microsoft/FeatureManagement-Dotnet) in .NET.

This post primarily talks about one way to implement that.


## .NET API ##

.NET has extensive Feature Management API which is enabled via [Microsoft.FeatureManagement](https://docs.microsoft.com/en-us/dotnet/api/microsoft.featuremanagement?view=azure-dotnet-preview) namespace.

The important [constucts](https://docs.microsoft.com/en-us/azure/azure-app-configuration/concept-feature-management#basic-concepts) for Feature Management include:

- ***Feature Flag Definition:*** This is how feature flags are configured. It Includes several stores and ***IConfiguration*** providers  ***but not all support change notifications***. For my use case I leveraged standard appsettings.json and/or [Azure App Configuration](https://docs.microsoft.com/en-us/azure/azure-app-configuration/overview).

- ***Feature Manager:*** Logic for handling life cycle of all feature flags (esp. notifications when values change).

- ***Feature Filter:*** Definition / Rule when the Feature Flag should be enabled / disabled.  Some Feature Flags are binary on/off but some depend on additional rules / filters which are used to turn them on/off.

Now, there are already a [few](https://learn.microsoft.com/en-us/dotnet/api/microsoft.featuremanagement.ifeaturefilter?view=azure-dotnet-preview) pre-existing filters provided by the library and for my use case, [Time Window](https://learn.microsoft.com/en-us/dotnet/api/microsoft.featuremanagement.featurefilters.timewindowfilter?view=azure-dotnet-preview) fitted perfectly well. ***NOTE: In the next post we will see how we can implement our own custom filter.***

So with that in place, lets look at the sample implementation.


## Implementation ##

The source code of this is available [here](https://github.com/AdiThakker/FeatureManagement) and some of the intersting snippets are shown below:

### Feature Definition ###

Starting with the definition, The Time Window definition in the appsettings.json is straight out of [docs](). The only point to notice is that the {StartDate} and {EndDate} tokens are the dynamic values which get set in the CD pipeline (any token replacement) task.

~~~json
{
  "FeatureManagement": {
    "VerboseLogging": {
      "EnabledFor": [
        {
          "Name": "TimeWindow",
          "Parameters": {
            "Start": "{StartDate}",
            "End": "{EndDate}"
          }
        }
      ]
    }
  }
}

~~~

### Feature Evaluation ###

To assist with Feature evaluation, I have created some wrappers around the built-in API. These  classes mainly provide some additional overloads and ability to enable/disable Feature Flag definition on a ***class***, or a ***method***. 

#### FeatureDefinitionAttribute ####

This is a custom [Attribute](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/attributes/creating-custom-attributes) implementation to decorate class or method. ***The idea behind this is to allow for Feature Flag definition and exeuction at either a class or a method level.***  

~~~csharp

[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
public sealed class FeatureDefinitionAttribute : Attribute
{
    public string FeatureName { get; }

    public FeatureFlagDefaultState DefaultState { get; }

    public FeatureDefinitionAttribute(string name, FeatureFlagDefaultState flagDefaultState = FeatureFlagDefaultState.Unkown)
        => (FeatureName, DefaultState) = (name, flagDefaultState);

}

public enum FeatureFlagDefaultState : int
{
    Unkown = 0,
    Disabled = 1,
    Enabled = 2
}

~~~
***NOTE: The enum is just to store a default state of the feature flag.***

#### FeatureFlagManagement ####

This is the main class the Implements [IFeatureFlagManagement]() contract and adds convenient APIs around the built-in [FeatureFlagManager]()

~~~csharp
public class FeatureFlagManagement : IFeatureFlagManagement
{
    private readonly ILogger<FeatureFlagManagement> logger;

    private readonly IFeatureManager featureManager;

    public FeatureFlagManagement(ILogger<FeatureFlagManagement> logger, IFeatureManager featureManager)
        => (this.logger, this.featureManager) = (logger, featureManager);


    public async Task<TResult> ExecuteIfFeatureEnabledAsync<TResult>(Func<TResult> executeFunc)
    {
        FeatureDefinitionAttribute? flag = GetAttribute<Type>(executeFunc.GetType());
        if (await IsFeatureEnabledAsyncHelper<object>(flag!.FeatureName, flag.DefaultState, null))
            return executeFunc();
        else
            return default;
    }

    public async Task<TResult> ExecuteIfFeatureEnabledAsync<TResult, TContext>(Func<TResult> executeFunc, TContext context)
    {
        FeatureDefinitionAttribute? flag = GetAttribute<Type>(executeFunc.GetType());
        if (await IsFeatureEnabledAsyncHelper<object>(flag!.FeatureName, flag.DefaultState, context))
            return executeFunc();
        else
            return default;
    }

    public async Task<bool> IsFeatureEnabledAsync(string name) => await IsFeatureEnabledAsyncHelper<object>(name, FeatureFlagDefaultState.Unkown, null);

    public async Task<bool> IsFeatureEnabledAsync<TContext>(string name, TContext context) => await IsFeatureEnabledAsyncHelper<object>(name, FeatureFlagDefaultState.Unkown, context);

    public async Task<bool> IsFeatureEnabledAsync<TType>(TType instance) where TType : class
    {
        FeatureDefinitionAttribute? flag = GetAttribute<Type>(instance.GetType());
        return await IsFeatureEnabledAsyncHelper<object>(flag!.FeatureName, flag.DefaultState, null);
    }

    public async Task<bool> IsFeatureEnabledAsync<TType, TContext>(TType instance, TContext context) where TType : class
    {
        FeatureDefinitionAttribute? flag = GetAttribute<Type>(instance.GetType());
        return await IsFeatureEnabledAsyncHelper<object>(flag!.FeatureName, flag.DefaultState, context);
    }

    Task<TResult> IFeatureFlagManagement.ExecuteIfFeatureEnabledAsync<TResult>(Func<TResult> executeFunc)
    {
        throw new NotImplementedException();
    }

    private async Task<bool> IsFeatureEnabledAsyncHelper<TContext>(string feature, FeatureFlagDefaultState defaultState, TContext? context)
    {
        return context switch
        {
            null => await featureManager.IsEnabledAsync(feature),
            _ => await featureManager.IsEnabledAsync(feature, context)
        };
    }

    private FeatureDefinitionAttribute? GetAttribute<TResult>(Type executingType) => executingType.GetCustomAttributes(true).OfType<FeatureDefinitionAttribute>().FirstOrDefault();
}
~~~

in the above snippet, you can see that the main logic to check if the featureflag is enabled is done in ***IsFeatureEnabledAsyncHelper***. All other methods are just convenient wrappers to work with the constructs (classes, methods) that use ***FeatureDefinitionAttribute***


#### Startup ####


#### Running Example ####

