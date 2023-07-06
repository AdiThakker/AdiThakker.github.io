---
layout:     post
title:      Feature Flags in .NET - (Post 2)
date:       2023-07-04
summary:    This is our continuation in exploring Feature Flags in .NET series.
categories: .NET, Azure, Feature-Flags
---


Hello Folks! I have returned after a significant break and recently wrapped up a project. Therefore, I now have some spare time to resume blogging, allowing us to pick up from where we last left off 😊. Now, let's continue with this post, which serves as a continuation of our [previous]({{site.url}}/Feature-Flags-In-NET-1) discussion on Feature Flags in .NET.

If you have read through the previous post, you know that the essence of the Time Window feature evaluation is mainly in its [TimeWindowFeatureFilter](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/FeatureFilters/TimeWindowFilter.cs), where it evaluates [TimeWindowFilterSettings](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/FeatureFilters/TimeWindowFilterSettings.cs). Based on how its configured, if uses the built-in [ConfigurationFeatureDefinitionProvider](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/ConfigurationFeatureDefinitionProvider.cs) to construct its [FeatureDefinition](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/FeatureDefinition.cs). This is all very nice, as long as you are using one of the preexisiting filters. 

***But what if you want to implement your own Custom logic for a feature evaluation?***

That's where we enter [ContextualFeatureFilter](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/IContextualFeatureFilter.cs). By implementing this interface, you can customize logic for feature evaluation, and that's what we will look at in this post.

***NOTE: I have made updates to the source code since my previous post to utilize the ContextualFeatureFilter. Please make sure to refer to the latest code in the [repository](https://github.com/AdiThakker/FeatureManagement) for the most recent version.***

In the previous post, we already had implemented a feature called ***VerboseLogging***, which leveraged  [TimeWindow](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/FeatureFilters/TimeWindowFilter.cs) for its evaluation. In this example, we will add a new feature called ***AddressLogging***, which will log address, if it matches our configured criteria.

So let's get started.

## Configuration

Again, we leverage ***appSettings.json*** for configuration. 

~~~json
"FeatureManagement": {
    "VerboseLogging": {
      "EnabledFor": [
        {
          "Name": "TimeWindow",
          "Parameters": {
            "End": "01/08/2023 12:00:00"
          }
        }
      ]
    },
   "AddressLogging": {       // Feature name
      "EnabledFor": [
        {
          "Name": "PersonAddress",    // Filter Condition (without Filter suffix)
          "Parameters": {
            "Address": "Famous Street"
          }
        }
      ]
    }
  }  
~~~

The key thing to note here is that we have added a new feature called ***AddressLogging***, which is enabled for a filter called ***PersonAddress***. We will look at this filter implementation in the next section. 

## Custom Filter

~~~csharp
 public class PersonAddressFilter : IContextualFeatureFilter<Person>
{        
    public Task<bool> EvaluateAsync(FeatureFilterEvaluationContext featureFilterContext, Person appContext)
    {
        _ = featureFilterContext ?? throw new ArgumentNullException(nameof(featureFilterContext));
        _ = appContext ?? throw new ArgumentNullException(nameof(appContext));

        var printAddressFilterConfig = featureFilterContext.Parameters.Get<Person>();
        if(printAddressFilterConfig != null)
        {
            return Task.FromResult(string.Equals(printAddressFilterConfig.Address, appContext.Address, StringComparison.InvariantCultureIgnoreCase));
        }

        return Task.FromResult(false);
    }
}
~~~

Our [PersonAddressFilter](https://github.com/AdiThakker/FeatureManagement/blob/main/FeatureManagement.Console/FeatureManagement/Filters/PersonAddressFilter.cs) implements the [IContextualFeatureFilter](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/IContextualFeatureFilter.cs) and its ***EvaluateAsync*** method takes in a ***FeatureFilterEvaluationContext*** and a ***Person*** object. The [FeatureFilterEvaluationContext](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/FeatureFilterEvaluationContext.cs) has the context of the feature evaluation, which includes the ***Parameters*** (of type ***IConfiguration***) and the ***FeatureName***. The ***Person*** object is the object that we is used to evaluate our custom logic against. 

In the above case, if the person's address matches the address configured in the ***appSettings.json***, then the feature is enabled, otherwise it is disabled.


### Filter Registration

~~~csharp
public static class FeatureManagerExtensions
{
    public static IServiceCollection AddFeatureConfiguration(this IServiceCollection collection, Action<IFeatureManagementBuilder> addFeatureFilters = null)
    {
        var featureBuilder = collection.AddFeatureManagement().AddFeatureFilter<TimeWindowFilter>();

        // add additional feature filters if any
        addFeatureFilters?.Invoke(featureBuilder);

        return collection;
    }
}
~~~

Our custom extension [FeatureManagerExtensions](https://github.com/AdiThakker/FeatureManagement/blob/main/FeatureManagement.Console/FeatureManagement/FeatureManagerExtensions.cs) class, is used to register our filters. It's ***AddFeatureConfiguration*** method takes in an ***Action<IFeatureManagementBuilder>***, which is used to register additional filters. 

In our case, we will use it to register our ***PersonAddressFilter*** and that's shown below in our [Program](https://github.com/AdiThakker/FeatureManagement/blob/main/FeatureManagement.Console/Program.cs). You can see the call to ***AddFeatureFilter***, which takes in our ***PersonAddressFilter***.

~~~csharp
var app = Host.CreateDefaultBuilder()
    .ConfigureLogging((loggingBuilder) =>
    {
        loggingBuilder.AddConsole();
    })
    .ConfigureServices(services =>
    {
        services.AddFeatureConfiguration(_ => _.AddFeatureFilter<PersonAddressFilter>());
        services.AddTransient<IFeatureFlagManagement, FeatureFlagManagement>();

    })
    .Build();
~~~

### Filter Evaluation.

This part of code is modified from the previous post, I have moved the this logice to a separate [PersonExtensions](https://github.com/AdiThakker/FeatureManagement/blob/main/FeatureManagement.Console/PersonExtensions.cs) class as shown below.

~~~csharp
public static class PersonExtensions
{
    public static async Task<string> Display(this Person person, IFeatureFlagManagement featureManager) => await featureManager.IsFeatureEnabledAsync(person) switch
    {
        true => string.Concat("Verbose Logging enabled: ", person.Name, " - ", person.Address),
        _ => string.Concat("Verbose Logging disabled: ", person.Name)
    };

    public static async Task<string> CustomDisplay(this Person person, IFeatureFlagManagement featureManager) => await featureManager.IsFeatureEnabledAsync("AddressLogging", person) switch
    {
        true => string.Concat("Address Logging enabled: ", person.Name, " - ", person.Address),
        _ => string.Concat("Address Logging disabled: ", person.Name)
    };
}
~~~

You can see above that we have added a new ***CustomDisplay*** method, which takes in the ***IFeatureFlagManagement*** isntance and uses it to evaluate the ***AddressLogging*** feature. This is done by passing in the ***Person*** object and the ***FeatureName*** to the [FeatureFlagManagement](https://github.com/AdiThakker/FeatureManagement/blob/main/FeatureManagement.Console/FeatureManagement/FeatureFlagManagement.cs) ***IsFeatureEnabledAsync*** method.

### Running Sample

Following snippet from our [Program](https://github.com/AdiThakker/FeatureManagement/blob/main/FeatureManagement.Console/Program.cs) class shows how we use the ***IFeatureFlagManagement*** instance to evaluate the feature.

~~~csharp
var featureManager = app.Services.GetService<IFeatureFlagManagement>()!;

// Time Window logging feature
var person = new Person();
Console.WriteLine(await person.Display(featureManager));

// Address logging feature (disabled)
Console.WriteLine(await person.CustomDisplay(featureManager));

// Address logging feature (enabled)
person.Address = "Famous Street";
Console.WriteLine(await person.CustomDisplay(featureManager));
~~~

When we run the application, we see the following output.   

![Setup]({{site.url}}/images/fflags-3.png){:height="100px" width="500px"}


This completes our series on Feature Management. I hope you found this useful! We'll most likely revisit this topic in the future, esp. around [Azure App Configuration](https://docs.microsoft.com/en-us/azure/azure-app-configuration/overview) as I am sure there will be more to come in this space.