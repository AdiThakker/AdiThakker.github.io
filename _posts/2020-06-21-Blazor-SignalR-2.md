---
layout:     post
title:      Blazor with SignalR - Series (Post 2)
date:       2020-06-21
summary:    This is the continuation of the previous post. In this post we will see how to setup a timer in Blazor using SignalR.  
categories: Blazor, SignalR, ASP.Net Core
---

In the [previous]() post, we saw how to get started with Blazor and customized it's default layout. This is shown here:

![Setup]({{site.url}}/images/Blazor-Signalr-11.png)

In this post we will continue to build on it and see how we can integrate a Timer using SignalR.

The first thing I did after changing the layout was to clean up the default Navmenu items and other razor, controller & shared components related to it. So the layout changed to this:

![Setup]({{site.url}}/images/Blazor-Signalr-13.png)

***Note:*** Blazor uses [open iconic](https://useiconic.com/open) icons sets so I changed the Navmenu link to show the correct one as shown here

~~~HTML
<NavLink class="nav-link text-dark" href="" Match="NavLinkMatch.All">
    <span class="oi oi-timer" aria-hidden="true"></span> Timer
</NavLink>
~~~

OK, the next step was to add SignalR Nuget components. So I added Microsoft.AspNetCore.SignalR library to the Blazor-Web.server project.

If you have read about [SignalR](https://docs.microsoft.com/en-us/aspnet/core/signalr/introduction?view=aspnetcore-3.1) then you know that is uses Hub as a central component to communicate betwen clients & servers. So in the Blazor-Web.Server project, I added ***ClientHub*** class and inherited from the ***Hub*** class as shown here:

~~~csharp

~~~


For the Timer functionality, the first thing I did was add a ***Timer*** class in the Blazor-Web.Server project. This class will encapsulate all the logic for sending all the 







In, the next post we will explore how to integrate SignalR for the timer functionality.