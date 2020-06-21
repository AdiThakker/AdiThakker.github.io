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

***Note:*** Blazor uses [open iconic](https://useiconic.com/open) icons sets so I changed the Navmenu link to show the new icon for the Timer one as shown here:

~~~HTML
<NavLink class="nav-link text-dark" href="" Match="NavLinkMatch.All">
    <span class="oi oi-timer" aria-hidden="true"></span> Timer
</NavLink>
~~~

OK, the next step was to add SignalR Nuget components. So I added Microsoft.AspNetCore.SignalR library to the BlazorWeb.server project.

If you have read about [SignalR](https://docs.microsoft.com/en-us/aspnet/core/signalr/introduction?view=aspnetcore-3.1) then you know that is uses Hub as a central component to communicate betwen clients & servers. So in the Blazor-Web.Server project, I added ***ClientHub*** class and inherited from the ***Hub*** class as shown here:

~~~csharp
namespace BlazorWeb.Server.Hubs
{
    public class ClientHub : Hub
    {
        private readonly TimerService _timerService;

        public ClientHub(TimerService timerService) => _timerService = timerService;

        public async Task Start() => await Task.Run(() => _timerService.Start());

        public async Task Stop() => await Task.Run(() => _timerService.Stop());
    }
}
~~~

You can see that the ***ClientHub*** references ***TimerService*** class which encapsulates the timer logic. The other Start & Stop methods just delegate the functionality to that Timer class's Start & Stop methods.

OK, so the next step was to build the ***Timer*** class which is shown below: you can see that the ***ClientHub*** is inject

For the Timer functionality, the first thing I did was add a ***Timer*** class in the Blazor-Web.Server project. This class will encapsulate all the logic for sending all the 

***Note:*** You can probably use the Controller class to achieve the same functionality of the TimerService class.

~~~csharp
 namespace BlazorWeb.Server.Services
{
    public class TimerService
    {
        private Timer _timer;
        private IHubContext<ClientHub> _hub;

        public TimerService(IHubContext<ClientHub> hub)
        {
            if (_timer == null)
            {
                _timer = new Timer(1000);
                _timer.Elapsed += Timer_Elapsed;
                _hub = hub;
            }
        }

        private void Timer_Elapsed(object sender, ElapsedEventArgs e)
        {
            if (_hub != null)
                _hub.Clients.All.SendAsync("ReceiveTime", DateTime.Now.ToString());
        }

        public void Start() => _timer.Start();

        public void Stop()
        {
            _timer.Stop();

            if (_hub != null)
                _hub.Clients.All.SendAsync("StopTime");
        }
    }
}
~~~





In, the next post we will explore how to integrate SignalR for the timer functionality.