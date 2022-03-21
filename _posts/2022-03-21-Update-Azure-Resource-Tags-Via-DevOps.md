---
layout:     post
title:      Update Azure Resource Tags via DevOps
date:       2022-03-21
summary:    This post explores how to update Azure Function Tags via Azure Devops leveraging Powershell.
categories: Azure DevOps, Powershell 
---

In a recent development at work, we were looking for a way to expose certain metadata into the Azure Portal. This came out of that fact that we were trying to troubleshoot an issue with our functions, only to find out that one of the dependencies was not updated to the latest version.

So that thought came was, if we exposed / surfaced up that dependency version and other related information on the Portal, we could have simplified that troubelshooting effort and therefore we decided to export that metadata to the portal via [Azure Resource Tags]()

In this post we will explore how we did this:

Like any other devops process, we were leveragining Azure CI/CD pipelines for deploying our Azure Functions, 

***NOTE: We were still using the [Classic]() instead of the [code]() format.So 2e started with [Powershell]() task***

![image]({{site.url}}/images/classes-et-1.png)

~~~powershell

~~~

















