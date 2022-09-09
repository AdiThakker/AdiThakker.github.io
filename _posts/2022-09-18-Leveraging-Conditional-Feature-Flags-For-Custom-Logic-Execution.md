---
layout:     post
title:      Leveraging Conditional Feature Flags for Custom Logic Execution
date:       2022-09-18
summary:    This post explores how to leverage conditional Feature Flags for executing any custom logic in your .NET code
categories: .NET, Azure
---

## Feature Management ##

[Feature Toggles / Feature Management](https://docs.microsoft.com/en-us/azure/azure-app-configuration/concept-feature-management) allows you to enable / disable certain features of your application dynamically. This can be very useful in certain scenarios such as:

- Restricting specific features to only certain set of users.

- Avoiding rollback or immediate hotfix by quickly deactivating a problem feature.

- Stabilize your application by turning off optional features during peak usage.

- Testing in production with only limited users before releasing it (sometimes a slippery slope 😉).


## .NET API ##

Feature Management in .NET is enabled via [Microsoft.FeatureManagement](https://docs.microsoft.com/en-us/dotnet/api/microsoft.featuremanagement?view=azure-dotnet-preview) namespace.

The [constucts](https://docs.microsoft.com/en-us/azure/azure-app-configuration/concept-feature-management#basic-concepts) that are key to Feature Management include:

- Feature Flag Definition: The repository where these feature flags are configured. Can include standard appsettings.json or [Azure App Configuration]()

- Feature Manager: Logic for handling life cycle of all feature flags

- Feature Filter: A rule for evaluating the state of a feature flag. 

So lets get started and see how we can leverage a built-in [TimeWindow] filter to execute some custom logic.

(NOTE:) In the next post we will see how we can implement our own custom filter.

## Implementation ##

The source code of this is available [here](). So lets start looking at some of the intersting snippets.

### Feature Definition ###


### Feature Evaluation ###




