---
layout:     post
title:      SignalR Integration with APIM - Real-time Communication (Post 1)
date:       2023-09-15
summary:    In this post, we will see how to integrate SignalR WebSocket endpoint in Azure APIM. 
categories: Azure, WebSockets, SignalR, APIM
---

If you have dabbled in the world of real-time communication in web applications, then you've probably stumbled upon SignalR - a powerful library that adds real-time web functionality to ASP.NET applications. However, managing and scaling these SignalR apps can become quite a challenge. Enter Azure API Management (APIM) - a tool that lets you publish, manage, secure, and analyze your APIs in a few simple steps.

Now, imagine the power of integrating SignalR WebSocket endpoints into APIM. This would mean handling numerous real-time communications at scale, with the robust security and flexibility that APIM offers.

***NOTE: In this first post, we will be walking through the process of deploying these resources using IaC. 
The next post will cover  integrating SignalR WebSocket endpoint in APIM.***

## APIM, SignalR and Bicep 

[APIM](https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts) provides a unified API gateway for your backend services, irrespective of whether these services are hosted on Azure, on-premises or hosted in other clouds. [SignalR](https://learn.microsoft.com/en-us/aspnet/signalr/overview/getting-started/introduction-to-signalr), on the other hand, simplifies the process of adding real-time web functionality to applications, enabling them to send push notifications to the client-side in real-time. Lastly, [Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview?tabs=bicep) (IaC) is a declarative language for describing and deploying Azure resources. It provides a transparent abstraction over ARM and makes it easier to orchestrate Azure services.

The combination of these three is incredibly powerful. With APIM, we have a reliable, secure platform for managing APIs. SignalR allows our application to communicate in real-time and IaC allows us to reliably and consistently deploy our application. Now let's dive into the implementation.

## Implementation

You can find the source code of this [here](https://github.com/AdiThakker/ApimIaC/tree/main/Deployment). Let's go through some of the key parts.

### Creating resources in Azure

For this, we used [modules](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/modules) in Bicep. 

Modules in Bicep allow for the encapsulation of sets of Azure resources. It's a way to reuse and share the code for similar deployments. 

#### APIM Instance

Our [APIM](https://github.com/AdiThakker/ApimIaC/blob/main/Deployment/modules/apim.bicep) module looks like this:

~~~javascript
param location string
param resourceNamePrefix string

resource apim 'Microsoft.ApiManagement/service@2021-01-01-preview' = {
  name: '${resourceNamePrefix}-apim'
  location: location
  properties: {
    publisherEmail: 'admin@${resourceNamePrefix}-apim.com'
    publisherName: '${resourceNamePrefix} API Management'
    sku: {
      name: 'Developer'
      capacity: 1
    }
  }
}

output apimName string = apim.name
~~~

#### SignalR Instance

Similary, we setup SignalR using our [Signalr](https://github.com/AdiThakker/ApimIaC/blob/main/Deployment/modules/signalr.bicep) resource module as shown below:

~~~javascript
param location string
param resourceNamePrefix string

resource signalr 'Microsoft.SignalRService/signalR@2022-08-01-preview' = {
  name: '${resourceNamePrefix}-signalr'
  location: location
  sku: {
    name: 'Standard_S1'
  }
}

output signalrName string = signalr.name

~~~

Both the APIM and SignalR modules are bundled and orchestrated in a master Bicep file, which we called [main.bicep](https://github.com/AdiThakker/ApimIaC/blob/main/Deployment/main.bicep). This main file ensures that the necessary dependencies between resources are maintained and this is what orchestrates our deployment.

~~~javascript
param location string = 'westus'
param resourceNamePrefix string = 'apim_resource'

module apim './modules/apim.bicep' = {
  name: 'apim'
  params: {
    location: location
    resourceNamePrefix: resourceNamePrefix
  }
}

module signalr './modules/signalr.bicep' = {
  name: 'signalr'
  params: {
    location: location
    resourceNamePrefix: resourceNamePrefix
  }
}
~~~

### Azure DevOps

For automating the deployment of these resources, we integrated Bicep with Azure DevOps. We defined our CI/CD process in an [azure-pipelines.yml](https://github.com/AdiThakker/ApimIaC/blob/main/Deployment/azure-pipelines.yml) file, which ensures that our infrastructure gets deployed as code every time there are changes.

~~~yaml
trigger:
  - main
  
pool:
  vmImage: 'ubuntu-latest'

stages:
- template: pipeline-templates/deploy.yml
parameters:
  environment: 'dev'
~~~

The above pipeline uses [templates](https://github.com/AdiThakker/ApimIaC/tree/main/Deployment/pipeline-templates) for resuability, which looks like this:

~~~yaml
parameters:
  - name: environment
    displayName: Environment
    type: string
    default: dev
    values:
      - dev
      - staging
      - prod

stages:
- stage: DeployResources_${{parameters.environment}}
  displayName: Deploy Resources - {{parameters.environment}}
  pool:
    vmImage: 'ubuntu-latest'
    
~~~

### Azure CLI

If you prefer using Azure CLI for deploying resources, you can manually provision the required resources using Bicep and Azure CLI as shown in the [Read Me](https://github.com/AdiThakker/ApimIaC/blob/main/README.md)

Alright Folks! That's all for now. In the next post, we will see how to integrate SignalR WebSocket endpoint in APIM. Stay tuned!
