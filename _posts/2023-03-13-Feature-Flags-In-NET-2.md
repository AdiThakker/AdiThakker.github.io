---
layout:     post
title:      Feature Flags in .NET - (Post 2)
date:       2023-07-04
summary:    This is our continuation in exploring Feature Flags in .NET series.
categories: .NET, Azure, Feature-Flags
---


Hello Folks! I have returned after a significant break and recently wrapped up a project. Therefore, I now have some spare time to resume blogging, allowing us to pick up from where we last left off 😊. Now, let's continue with this post, which serves as a continuation of our [previous]({{site.url}}/Feature-Flags-In-NET-1) discussion on Feature Flags in .NET.

If you have read through the previous post, you know that the essence of the Time Window feature evaluation is mainly in its [TimeWindowFeatureFilter](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/FeatureFilters/TimeWindowFilter.cs), where it evaluates [TimeWindowFilterSettings](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/FeatureFilters/TimeWindowFilterSettings.cs) based on how its comfigured using the built-in [ConfigurationFeatureDefinitionProvider](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/ConfigurationFeatureDefinitionProvider.cs) to construct its [FeatureDefinition](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/FeatureDefinition.cs).


This is all very nice, as long as you are using one of the preexisiting filters. But what if you want to implement your own Custom logic for a feature evaluation? That's where we enter [ContextualFeatureFilter](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/IContextualFeatureFilter.cs) and that's what we will look at in this post.

***NOTE: I have made updates to the source code since my previous post to utilize the ContextualFeatureFilter. Please make sure to refer to the latest code in the repository for the most recent version.***

We already had implemented a feature called ***VerboseLogging***, which leveraged  [TimeWindow]() for its evaluation. In this example, we will add a new feature called ***AddressLogging*** and specify custom logic for its evaluation .i.e  we will print address, if the name matches our specified criteria.

So let's get started.

## Configuration.

appSettings now includes a new feature definition, called AddressLogging, which is enabled when the Address matches our specified criteria.

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

## Custom filter.

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

you can see above that our ***PersonAddressFilter*** implements the [IContextualFeatureFilter]() and its ***EvaluateAsync*** method takes in a ***FeatureFilterEvaluationContext*** and a ***Person*** object. The ***FeatureFilterEvaluationContext*** is the what has the context of the feature evaluation, which includes the ***Parameters*** (which are of type ***IConfiguration***) and the ***FeatureName***. The ***Person*** object is the object that we will use to evaluate our custom logic against.


### Filter Registration.

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

***FeatureManagementExtensions*** is an extension class that we use to register our filters. It's ***AddFeatureConfiguration*** method takes in an ***Action<IFeatureManagementBuilder>***, which is used to register additional filters. In our case, we will use it to register our ***PersonAddressFilter***.
This is shown below, in the ***Program***, where we register our ***PersonAddressFilter*** using the ***AddFeatureFilter*** method.

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

You can see above that we have added a new ***CustomDisplay*** method, which takes in the ***IFeatureFlagManagement*** and uses it to evaluate the ***AddressLogging*** feature. This is done by passing in the ***Person*** object and the ***FeatureName*** to the ***IsFeatureEnabledAsync*** method.

### Run the application.

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


