---
layout:     post
title:      Mermaid, Azure DvOps Wiki and Sequence Diagrams
date:       2020-10-11
summary:    In this post, we will see how we can use Mermaid tool in Azure Devops to generate Sequence Diagrams  
categories: Azure Devops, Mermaid
---

Hello Folks! It's been a while since I have blogged 😊, just have been busy with client projects.

In the meantime, I have been thinking about some of the topics that I would like to share and one of them is using Mermaid in Azure Devops to generate sequence diagrams.

In several of my projects, we have been using Sequence Diagrams to show data flow between the architectural components and have mainly used [Web Sequence Diagrams](https://www.websequencediagrams.com/) for it. But sometime back on one of my projects, I came to know about [mermaid](http://mermaid-js.github.io/mermaid/) tool and found it pretty interesting. You can see that its a nifty tool that not only generates several types of diagrams but also integrates well with Azure DevOps Wiki Pages.

In order to get started with this here are the steps:

**Note:** You will need publish code a wiki feature to be able to use this feature. More details [here](https://docs.microsoft.com/en-us/azure/devops/project/wiki/publish-repo-to-wiki?view=azure-devops&tabs=browser)

Once you have this enabled, create a new project in Azure Devops as shown here:

![Setup]({{site.url}}/images/mermaid-1.png)

We'll keep this setup simple so, select the Wiki tab and create project wiki.

![Setup]({{site.url}}/images/mermaid-2.png)

![Setup]({{site.url}}/images/mermaid-3.png)

If you search on-line for various [reasons](https://firstsiteguide.com/benefits-of-blogging/) why people blog, the ones that are most relevant to me are:
