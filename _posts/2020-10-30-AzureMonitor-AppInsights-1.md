---
layout:     post
title:      Azure Monitor and Application Insights - Series (Post 1)
date:       2020-10-30
summary:    In this first post, we will explore how we can get started with Application Insights and integrate that into our ASP Net Core Web Application.
categories: Azure Monitor, Application Insights
---

Instrumentation in your applications is very critical as it helps you understand how your applications are performing and proactively identifies issues affecting them and the resources they depend on.

Azure provies you with [Azure Monitor](https://azure.microsoft.com/en-us/services/monitor/) & [Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview) tools to help implement a good instrumentation strategy in your applications.

![Setup]({{site.url}}/images/AppInsights-1.png)

There's lots you can do within Azure Monitor as shown above but for our post we will just focus on the Application Insights part. 

Application Insights itself lets you monitor lots metrics such as Request rates, Response times, Failure rates, Exceptions, Usage tracking, etc (you can refer to the previous link for the comprehensive list)

OK, So how do we get started:

**First, Create Application Insights Resource**

Log into your Azure subscription, select Application Insights as the resource and create an instance as shown below:

![Setup]({{site.url}}/images/AppInsights-2.png)

Once provisioned, navigate to the resource and copy the instrumentation key as we will need this in the next step.

![Setup]({{site.url}}/images/AppInsights-3.png)

**Next, Enable Application Insights in your .Net core Web App**

First step to enable Application Insights is to download the appropriate NuGet pacakage as shown here:

![Setup]({{site.url}}/images/AppInsights-4.png)

Then, configure the startup.cs class ConfigureServices method to include the telemetry as shown here:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // The following line enables Application Insights telemetry collection.
    services.AddApplicationInsightsTelemetry();

    services.AddControllers();
}
```

Include the instrumentation key in the appsettings.json file as shown here:

```json
{
  "ApplicationInsights": {
    "InstrumentationKey": "02d1368d-c1f9-407b-bf5e-c67b59c1202e"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

**Monitor**

Loading the default Webforecast data automatically loads the following data out of the box as shown below

![Setup]({{site.url}}/images/AppInsights-5.png)

If you click on the server request chart, you can see more information showing response for the requests:

![Setup]({{site.url}}/images/AppInsights-6.png)