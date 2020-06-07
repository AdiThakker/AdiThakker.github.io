---
layout:     post
title:      Jekyll on Windows 10 with WSL 2
date:       2020-06-07
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

Next was installing [Ubuntu](https://www.microsoft.com/en-us/p/ubuntu/9nblggh4msv6?activetab=pivot:overviewtab) from Windows Store, setting up username, password, update default version to 2 and quick verification:
![Setup]({{site.url}}/images/WSL-verify.png)

I already had VS Code and git installed on my laptop so i went ahead with VS Code update and launched it from command line as indicated [here](https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-vscode) which installed VS Code server and then launched it. Once VS Code started, it prompted me to install Remote - WSL extension, which I proceeded with. This extension enables you to use WSL as your full-time development environment directly from VS Code.

OK, so the next step was to clone my repo and start installing Jekyll.


