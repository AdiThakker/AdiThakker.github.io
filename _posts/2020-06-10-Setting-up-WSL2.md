---
layout:     post
title:      Jekyll on Windows 10 with WSL 2
date:       2020-06-10
summary:    I wanted to install Jekyll on my personal laptop running Windows 10 for blogging and this post explores how I enabled that via WSL 2 setup. 
categories: Blogging, Ubuntu, WSL 2, Jekyll, Windows 10
---

So far I have used my office laptop for blogging and that has WSL installed (as the Windows version is controlled by corporate IT). I had a personal laptop sitting around, so I decided to use that for my blog using WSL 2. 

The first thing I wanted to download was the new [Windows Terminal](https://www.microsoft.com/en-us/p/windows-terminal/9n0dx20hk701), so I headed to the Windows Store and got it. ***Note:*** If you would like to customize the terminal, there are some nice instructions on how to do it by Scott Hanselman [here](https://www.hanselman.com/blog/HowToMakeAPrettyPromptInWindowsTerminalWithPowerlineNerdFontsCascadiaCodeWSLAndOhmyposh.aspx).

Following is my Sorin theme:
![Setup]({{site.url}}/images/Terminal-Theme.png)

The instructions on getting [started](https://docs.microsoft.com/en-us/windows/wsl/install-win10) with WSL 2 are very well documented and so I started there.

Firstly, I updated my Windows version to the one that supports WSL 2, shown below:
![Setup]({{site.url}}/images/Win-ver.png)

As instructed, then I enabled Windows Subsystem for Linux as shown below:
```Powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```
And then enabled Virtual Machine Platform by running:
```Powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```
After the restart, when I tried to set WSL default version to 2, I got a message stating ***WSL 2 requires an update to its kernel component***, so I proceeded with the update as shown below by downloading it from [here](https://docs.microsoft.com/en-us/windows/wsl/wsl2-kernel)
![Setup]({{site.url}}/images/WSL-kernel.png)

Next was installing [Ubuntu](https://www.microsoft.com/en-us/p/ubuntu/9nblggh4msv6?activetab=pivot:overviewtab) from Windows Store, setting up username, password and 



First i had to update my Windows version to Well getting WSL 2 installed meant i had to download the latest version of  I added archives page to my blog which was displaying correctly on my machine, however when I pushed changes to github, it failed. 

Inquiring further, I found out that the [plugin](https://jekyll.github.io/jekyll-archives/) which I used, is not supported by Github.

![Setup]({{site.url}}/images/Installing-archives-setup-2.png){:height="300px" width="800px"}

So I had to find an alternative solution and thanks to [this](https://dinobansigan.com/posts/adding-archive-page-to-jekyllnow-blog), I could get it working with just couple tweaks.

Firstly, I updated my ***year-archive.html*** with the following (similar to the one posted in the above link):

 {% raw %}
 ~~~html
<article class="page">
  <h1>{{ page.title }}</h1>
  <div class="entry">    
    {% assign previousYear = "" %}
    {% for post in site.posts %}
      {% capture currentYear %}
        {{ post.date | date: "%Y" }}
      {% endcapture %}
    
      {% if currentYear != previousYear %}
        {% assign previousYear = currentYear %}
        <h3>{{ currentYear }}</h3>
      {% endif %}
    
      {{ post.date | date: '%B %d, %Y' }} - <a style="font-weight: bold" href="{{ post.url }}">{{ post.title }}</a>
      <br />
    {% endfor %}    
  </div>
</article>
~~~
{% endraw %}

And removed all the plugin references from Gemfile, _config.yml. That's It!!!