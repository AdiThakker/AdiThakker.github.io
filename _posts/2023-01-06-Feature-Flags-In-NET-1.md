---
layout:     post
title:      Feature Flags in .NET - (Post 1)
date:       2023-01-06
summary:    Happy New Year, Folks!!! In this first post, we will explore how to implement conditional Feature Flags for executing any logic in your .NET code.
categories: .NET, Azure, Feature-Flags
---
 
## Feature Management ##

If you are aware of [Feature Toggles / Feature Management](https://docs.microsoft.com/en-us/azure/azure-app-configuration/concept-feature-management), then you know that it allows you to enable / disable features of your application dynamically. 

Such functionality can be very handy as you can control aspects such as:

- Restricting specific features to only certain set of users.

- Avoiding rollback or immediate hotfix by quickly deactivating a problem feature.

- Stabilize your application by turning off optional features during peak usage.

- Testing in production with only limited users. (sometimes a slippery slope 😉).

In the project that I was involved in recently, there was a requirement, where we had to enable ***Verbose Logging***, after the application was deployed and then automatically disable that after certain configured ***Time-Window*** (~ 24 hours) had passed. So this led me to explore the Feature Flags path and the [Feature Management API](https://github.com/microsoft/FeatureManagement-Dotnet) in .NET.

This post primarily talks about an implementation of that.


## .NET API ##

.NET has extensive Feature Management API which is enabled via [Microsoft.FeatureManagement](https://docs.microsoft.com/en-us/dotnet/api/microsoft.featuremanagement?view=azure-dotnet-preview) namespace.

The important [constucts](https://docs.microsoft.com/en-us/azure/azure-app-configuration/concept-feature-management#basic-concepts) for Feature Management include:

- ***Feature Flag Definition:*** This is how feature flags are configured. It Includes several stores and ***IConfiguration*** providers  ***but not all support change notifications***. For my use case I leveraged standard appsettings.json and [Azure App Configuration](https://docs.microsoft.com/en-us/azure/azure-app-configuration/overview) 
  
    ***Note: The sample implementation included here, does not demo Azure App Configuration, we ended up using that in the actual prod. scenario***.

- ***Feature Manager:*** Logic for handling life cycle of all feature flags (esp. notifications when values change).

- ***Feature Filter:*** Definition / Rule when the Feature Flag should be enabled / disabled.  Some Feature Flags are binary on/off but some depend on additional rules / filters which are used to turn them on/off.

Now, there are already a [few](https://learn.microsoft.com/en-us/dotnet/api/microsoft.featuremanagement.ifeaturefilter?view=azure-dotnet-preview) filters provided by the library and for my use case, [Time Window](https://learn.microsoft.com/en-us/dotnet/api/microsoft.featuremanagement.featurefilters.timewindowfilter?view=azure-dotnet-preview) fitted perfectly well. 

So with that in place, lets look at the sample implementation.


## Implementation ##

The simplified source code of this is available [here](https://github.com/AdiThakker/FeatureManagement) and some of the intersting snippets are shown below:

### Feature Definition ###

Starting with the definition, The Time Window definition in the appsettings.json is straight out of [docs](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/README.md#Feature-Flag-Declaration). The only point to notice is that the  {EndDate} token is a value which gets set in the CD pipeline (any token replacement) task.

~~~json
{
  "FeatureManagement": {
    "VerboseLogging": {
      "EnabledFor": [
        {
          "Name": "TimeWindow",
          "Parameters": {
            "End": "{EndDate}" // Token gets replaced with actual value during deployment
          }
        }
      ]
    }
  }
}

~~~

### Feature Evaluation ###

To assist with Feature evaluation, I have created some wrappers around the built-in API. These  classes essentially provide additional overloads with the ability to enable/disable Feature Flag definition on either a ***class***, or a ***method***. 

They are as follows: 

 - #### FeatureDefinitionAttribute ####

    This is a custom [Attribute](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/attributes/creating-custom-attributes) implementation to decorate class or method. ***You can isolate a Feature Flag behavior at either a class or a method by using this attribute.***  

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

 - #### FeatureFlagManagement ####

    This is the main class the Implements [IFeatureFlagManagement](https://github.com/AdiThakker/FeatureManagement/blob/main/FeatureManagement.Console/FeatureManagement/Interfaces/IFeatureFlagManagement.cs) contract and adds convenient APIs around the built-in [FeatureManager](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/FeatureManager.cs)

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

    In the above snippet, you can see that the main logic to check if the feature flag is enabled is done in ***IsFeatureEnabledAsyncHelper*** which delegates the check to built-in FeatureManager. 

    All other methods are just convenient wrappers to work with ***FeatureDefinitionAttribute*** shown earlier. 

  - #### FeatureManagerExtensions ####

    Feature registration is done via [FeatureManagerExtensions](https://github.com/AdiThakker/FeatureManagement/blob/main/FeatureManagement.Console/FeatureManagement/FeatureManagerExtensions.cs) class as shown below:

    ~~~csharp
    public static class FeatureManagerExtensions
    {
        public static IServiceCollection AddFeatureConfiguration(this IServiceCollection collection)
        {
            collection.AddFeatureManagement().AddFeatureFilter<TimeWindowFilter>();
            return collection;
        }
    }
    ~~~

### Running Sample ###

The [Person](https://github.com/AdiThakker/FeatureManagement/blob/main/FeatureManagement.Console/Person.cs) class shown below is where we define the ***Verbose Logging*** feature and its ***Display*** method is where the Feature is evaluated. 

~~~csharp
[FeatureDefinition("VerboseLogging")]
public class Person
{
    private IFeatureFlagManagement featureManager;

    public string Name => "John Doe";

    public string Address => "123 Street";

    public Person(IFeatureFlagManagement featureFlagManager) => featureManager = featureFlagManager;

    public async Task<string> Display() => await featureManager.IsFeatureEnabledAsync(this) switch
    {
        true => string.Concat("Verbose Logging enabled: ", Name , " ", Address),
        _ => string.Concat("Verbose Logging disabled: ", Name)
    };
}
~~~

Following is the output showing that in action,  you can see that when  ***Verbose Logging*** feature is enabled, both the name and the address are displayed, whereas only the name is displayed when that feature is disabled. 

![Setup]({{site.url}}/images/fflags-2.png){:height=100px" width="500px"}

![Setup]({{site.url}}/images/fflags-1.png){:height="100px" width="500px"}


So, there you see folks, In this post we saw how to leverage FeatureFlags in .NET. 

***In the next post we will dive deep into the workings behind it when we implement our own custom filter.***