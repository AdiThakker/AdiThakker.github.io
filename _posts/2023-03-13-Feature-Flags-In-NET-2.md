---
layout:     post
title:      Feature Flags in .NET - (Post 2)
date:       2023-07-04
summary:    This is our continuation in exploring Feature Flags in .NET series.
categories: .NET, Azure, Feature-Flags
---


Hello Folks! I am back after a long haitus, just completed a project, so now have some time to blog and so we can continue where we had left off 😊. Alright, so this post is continuation of our [previous]({{site.url}}/Feature-Flags-In-NET-1) one, where we saw how to get started with Feature Flags in .NET.

If you have read through the previous post, you'll see that the essence of the Time Window feature evaluation is mainly in its [TimeWindowFeatureFilter](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/FeatureFilters/TimeWindowFilter.cs), where it evaluates [TimeWindowFilterSettings](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/FeatureFilters/TimeWindowFilterSettings.cs) based on how its comfigured using the built-in [ConfigurationFeatureDefinitionProvider](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/ConfigurationFeatureDefinitionProvider.cs) to construct its [FeatureDefinition](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/FeatureDefinition.cs).


This is all very nice, as long as you are using one of the preexisiting filters. But what if you want to implement your own Custom logic for a feature evaluation, that's where we enter [ContextualFeatureFilter](https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/src/Microsoft.FeatureManagement/IContextualFeatureFilter.cs) and that's what we will look at in this post.


In this example, we will modify our earlier example of logging feature evaluation from [TimeWindow]() to our own Custom logic .i.e instead of printing address, when timewindow is enabled, we will print address, if the name matches our specified criteria.

So let's get started.

***NOTE: Some of the source code has chanaged to take the benefit of ContextualFeatureFilter, so please refer to the source code in the repo for the latest code.***

The first step needed to implement CustomFilter is to implement the []() and following is the definition of our class






