---
layout:     post
title:      Mermaid, Azure DevOps Wiki and Sequence Diagrams
date:       2020-10-18
summary:    In this post, we will see how we can use Mermaid tool in Azure DevOps Wiki to generate Sequence Diagrams  
categories: Azure Devops, Mermaid
---

Hello Folks! It's been a while since I have blogged 😊, just have been busy with client projects.

In the meantime, I have been thinking about some of the topics that I would like to share and one of them is using Mermaid syntax in Azure Devops Wiki pages to generate Sequence diagrams.

In several of my projects, we have been using Sequence diagrams to show data flow between the architectural components and have mainly used [Web Sequence Diagrams](https://www.websequencediagrams.com/) for it. But sometime back on one of my projects, I came to know about [mermaid](http://mermaid-js.github.io/mermaid/) tool and found it pretty interesting. You can see that its a nifty tool that not only generates several types of diagrams as code but also integrates well with Azure DevOps Wiki Pages.

Lets look at how do we go about doing this:

**Creating Azure Devops Wiki**

**Note:** You will need the right version of Azure DevOps for this integration. For more details refer [here](https://docs.microsoft.com/en-us/azure/devops/project/wiki/wiki-markdown-guidance?view=azure-devops)

Once you have this verified, create a new project in Azure DevOps as shown here:

![Setup]({{site.url}}/images/mermaid-1.png)

We'll keep this setup simple so, select the Wiki tab and create project wiki.

![Setup]({{site.url}}/images/mermaid-2.png)

Then, enter the title and click on the Insert Mermaid Diagram, which will generate a sample graph diagram

![Setup]({{site.url}}/images/mermaid-3.png)

![Setup]({{site.url}}/images/mermaid-4.png)

**Understanding mermaid markdown syntax**

[Mermaid](https://mermaid-js.github.io/mermaid/getting-started/n00b-gettingStarted.html) website has good documentation around how to get started with different diagram types and they also have a [live editor](https://mermaidjs.github.io/mermaid-live-editor/#/edit/eyJjb2RlIjoiZ3JhcGggVERcbkFbQ2hyaXN0bWFzXSAtLT58R2V0IG1vbmV5fCBCKEdvIHNob3BwaW5nKVxuQiAtLT4gQ3tMZXQgbWUgdGhpbmt9XG5DIC0tPnxPbmV8IERbTGFwdG9wXVxuQyAtLT58VHdvfCBFW2lQaG9uZV1cbkMgLS0-fFRocmVlfCBGW2ZhOmZhLWNhciBDYXJdXG4iLCJtZXJtYWlkIjp7InRoZW1lIjoiZGVmYXVsdCJ9fQ), so I'll let you explore that. For our example, we will look at the sequence diagram and the following example:

![Setup]({{site.url}}/images/mermaid-5.png)


```markdown
::: mermaid
sequenceDiagram
    participant Alice
    participant Bob
    Alice->>John: Hello John, how are you?
    loop Healthcheck
        John->>John: Fight against hypochondria
    end
    Note right of John: Rational thoughts<br/>prevail...
    John-->>Alice: Great!
    John->>Bob: How about you?
    Bob-->>John: Jolly good!
:::
```

**Note:** Any mermaid specific syntax should be between start ***::: mermaid*** and ***:::*** tags. From the above example, you can see that the syntax is pretty straighforward. Some quick tips below:

- **SequenceDiagram** indicates the diagram type.  
- **participant** tells which the components in the sequence diagram.
- **->>** and **- ->>** indicate synchronous & asynchronous flows.
- **loop** and **Note right of John** and self-explanatory.

You can refer [here](https://mermaid-js.github.io/mermaid/diagrams-and-syntax-and-examples/examples.html#basic-sequence-diagram) to learn more about the different diagrams types

So there you see it was very easy to get started with Mermaid and Azure DevOps Wiki and the best part is since this is all backed by git repo, you it can be versioned and setup as part of your CI pipeline.

![Setup]({{site.url}}/images/mermaid-6.png)