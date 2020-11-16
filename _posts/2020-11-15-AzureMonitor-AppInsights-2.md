---
layout:     post
title:      Azure Monitor and Application Insights - Series (Post 2)
date:       2020-11-15
summary:    In this second post, we will explore how we can enable custom metrics in Application Insights.
categories: Azure Monitor, Application Insights
---

In the [previous]({{site.url}}/AzureMonitor-AppInsights-1) post, we saw how we can get started with Application Insights and integrate that into our ASP Net Core Web Application.

In this continuation, we will add custom metrics to our Application Insights instrumentation and view them in Azure portal and the very first thing you will need for this is [TelemetryClient](https://docs.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics). So lets get started...

First we verify that we have the right version of ApplicationInsights package in the {AppName}.csproj file:

~~~xml
<ItemGroup>
  <PackageReference Include="Microsoft.ApplicationInsights" Version="2.15.0" />
  <PackageReference Include="Microsoft.ApplicationInsights.AspNetCore" Version="2.15.0" />
</ItemGroup>
~~~

Then we enable the right level of logging in the appsettings.json file for ApplicationInsights:

~~~json
{
  "ApplicationInsights": {
    "InstrumentationKey": "INSTRUMENTATION-KEY",
    "cloud_RoleName": "Weather Forecaster Service"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information",
      "ApplicationInsights": {
        "LogLevel": {
          "Default": "Information"
        }
      }
    }
  },
  "AllowedHosts": "*"
}
~~~

Next we verify that we have enabled Application Insights correctly in the ***ConfigureServices*** method:

~~~csharp
// This method gets called by the runtime. Use this method to add services to the container.
public void ConfigureServices(IServiceCollection services)
{
    // The following line enables Application Insights telemetry collection.
    services.AddApplicationInsightsTelemetry();

    services.AddControllers();
}
~~~

Now, TelemetryClient gives you a handy way to log  data via its several ***TrackXXX*** methods, one of its parameters is of type IDictionary<string,string> and we will leverage that by adding ***CorrelationId*** and ***ZipCode*** custom properties so we can analyze how many times weather is requested for a specific zip code (just to demonstrate a simple use case 😉).

So first we modify the constructor of the ***WeatherController.cs*** file to inject the telemetryclient as shown below:

~~~csharp
public WeatherForecastController(IOptions<TelemetryConfiguration> options, ILogger<WeatherForecastController> logger)
{
    _logger = logger;
    _telemetryClient = new TelemetryClient(options.Value);
}
~~~

Next, we add a helper method to attach custom properties as part of the request:

~~~csharp
private static Dictionary<string, string> AttachCustomTelemetry(Guid correlationId, string zipCode)
{
    var telemetryParameters = new Dictionary<string, string>
    {
        { "operation_Id", correlationId.ToString() },
        { "zip_code", zipCode }
    };
    return telemetryParameters;
}
~~~

Now we attach those custom properties to the ***TrackTrace*** method of the TelemetryClient as shown below:

~~~csharp
[HttpGet("{zipCode}")]
public IEnumerable<WeatherForecast> Get(string zipCode)
{
    // Log Requests simulating external call to  Weather Service with custom telemetry
    _telemetryClient.TrackTrace("Weather Forecast Request", SeverityLevel.Information, AttachCustomTelemetry(Guid.NewGuid(), zipCode));
    return WeatherService_GetWeatherForecast();
}
~~~

When requesting weatherforecast a few times and then viewing the request information in Azure Application Insights portal, you can see that our custom properties can be viewed as shown here:

![Setup]({{site.url}}/images/CustomMetrics-3.png)

Pretty Neat!!!

So as you an see it was very easy to leverage TelemetryClient to send custom data to Application Insights. In the next part we will see how you can enable logs and use Kusto to analyze data using these custom properties.

Meanwhile I would encourage you to explore [TelemetryClient](https://docs.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics#prep) further as we have just used one way of logging custom properties and there are other APIs that you could use take this further 😊.