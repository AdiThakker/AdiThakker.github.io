---
layout:     post
title:      Blazor with SignalR - Series (Post 3)
date:       2020-06-27
summary:    This is the continuation of the previous post. In this post we will see how to setup a authentication using Azure AD.  
categories: Blazor, SignalR, ASP.Net Core
---

In the [previous]({{site.url}}/Blazor-SignalR-2) post, we saw how to get use SignalR within in Blazor

This is the final post in this series an we will see how we can leverage Azure AD for user Authentication. 

1. Setting up Azure AD . creating Azure tenant
First step was to create  new tenant. Following are the instructions on how to get it [here](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-create-new-tenant)

- Creating new User
![Setup]({{site.url}}/images/Blazor-AAD-1.png)


2. Registering Apps (server side and client)
![Setup]({{site.url}}/images/Blazor-AAD-2.png)

3. Server side configuration
![Setup]({{site.url}}/images/Blazor-AAD-3.png)
![Setup]({{site.url}}/images/Blazor-AAD-4.png)
![Setup]({{site.url}}/images/Blazor-AAD-5.png)
![Setup]({{site.url}}/images/Blazor-AAD-6.png)

4. Client side configuration

![Setup]({{site.url}}/images/Blazor-AAD-7.png)
![Setup]({{site.url}}/images/Blazor-AAD-8.png)
![Setup]({{site.url}}/images/Blazor-AAD-9.png)
![Setup]({{site.url}}/images/Blazor-AAD-10.png)

Install Microsoft.AspNetCore.Authentication.AzureAD.UI NuGet package


~~~csharp
public void ConfigureServices(IServiceCollection services)
{

    services.AddControllersWithViews();
    services.AddRazorPages();
    services.AddSignalR();
    services.AddAuthentication(AzureADDefaults.BearerAuthenticationScheme)
        .AddAzureADBearer(options => Configuration.Bind("AzureAd", options));
    services.AddSingleton<TimerService>();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
        app.UseWebAssemblyDebugging();
    }
    else
    {
        app.UseExceptionHandler("/Error");
        // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
        app.UseHsts();
    }

    app.UseHttpsRedirection();
    app.UseBlazorFrameworkFiles();
    app.UseStaticFiles();

    app.UseRouting();

    app.UseAuthentication();
    app.UseAuthorization();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapRazorPages();
        endpoints.MapControllers();
        endpoints.MapHub<ClientHub>("/timer");
        endpoints.MapFallbackToFile("index.html");
    });
}
~~~

Server appsettings.json Azure AD configuration for server is

~~~json
"AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "Domain": "blazorweb.onmicrosoft.com",
    "TenantId": "dd3566b9-e2d3-4191-a43d-e26588af60de",
    "ClientId": "ca7a2acf-647d-4077-ac00-6837e146b8fb"
  }
~~~

Import following NuGet package for the client App

Microsoft.Authentication.WebAssembly.Msal

In the program.cs add the following 

~~~csharp
builder.Services.AddMsalAuthentication(options =>
{
    builder.Configuration.Bind("AzureAd", options.ProviderOptions.Authentication);
    options.ProviderOptions.DefaultAccessTokenScopes.Add("{SCOPE URI}");
});
~~~


and the following AzureAD configuration in the appsetttings.json

~~~json
"AzureAd": {
    "Authority": "https://login.microsoftonline.com/dd3566b9-e2d3-4191-a43d-e26588af60de",
    "ClientId": "f6fbbd51-e7fd-48ef-bbea-87b437e391e6",
    "ValidateAuthority": true
  }
~~~
