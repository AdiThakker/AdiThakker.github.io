---
layout:     post
title:      Azure Monitor and Application Insights - Series (Post 2)
date:       2020-11-12
summary:    In this second post, we will explore how we can enable custom metrics in Application Insights.
categories: Azure Monitor, Application Insights
---

In the [previous]({{site.url}}/AzureMonitor-AppInsights-1) post, we saw how we can get started with Application Insights and integrate that into our ASP Net Core Web Application.

In this continuation, we will add custom metrics to our Application Insights instrumentation and view them in Azure portal and the very first thing you will need for this is [Telemetry Client](https://docs.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics). So lets get started...

First we verify that we have the right version of ApplicationInsights package from the project.json file:

![Setup]({{site.url}}/images/CustomMetrics-2.png)

Then we enable the right level of logging in the appsettings.json file:

![Setup]({{site.url}}/images/CustomMetrics-1.png)

