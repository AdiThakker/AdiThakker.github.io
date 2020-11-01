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

With this basic setup, when you launch the default Webforecast controller a few times, data automatically gets collected in Application Insights resource in your azure subscription.

By default, you can see you can view Failed Requests, Server Response time, Server requests, Availability

![Setup]({{site.url}}/images/AppInsights-5.png)

You can drill down further by clicking on the individual chart, the following is when you drilled down on the server requests chart:

![Setup]({{site.url}}/images/AppInsights-6.png)

Clicking on the Applcation Map shows the following:

**Note: i renamed my Application Insights instance to the appriprite meaningful name**

![Setup]({{site.url}}/images/AppInsights-7.png)

As you can see, this is not bad at all with wuch minimal configuration. In the next post we will see how we can add custom metric logging to this and drill down into the numbers further.