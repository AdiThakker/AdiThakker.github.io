---
layout:     post
title:      SignalR Integration with APIM - Real-time Communication (Post 2)
date:       2023-10-15
summary:    Our continuation of how to integrate SignalR WebSocket endpoint in Azure APIM. 
categories: Azure, WebSockets, SignalR, APIM
---

Having introduced the concept of integrating SignalR with APIM and deploying resources using Bicep in the previous post, we'll now take the next step: the actual integration of SignalR WebSocket endpoints into APIM. This integration lets us harness the full power of real-time communication managed by Azure API Management (APIM).

## APIM and WebSockets

Azure API Management (APIM) supports WebSocket APIs, allowing full-duplex communication channels to be initiated over a single TCP connection. With SignalR integrated into APIM, we can streamline the process of setting up WebSocket connections, ensuring that real-time data flows smoothly between clients and the server.

So lets get started!

## SignalR and APIM Integration

1. Configuring APIM for WebSocket:
   
First to configure APIM for WebSocket, you need to create a new WebSocket API. The following [link](https://learn.microsoft.com/en-us/azure/api-management/websocket-api?tabs=portal#add-a-websocket-api) explains how to create a WebSocket API.

2. Azure Function App for SignalR:
   
### Function App Code
   
Next, we'll create an Azure Function that acts as a hub for our SignalR service. This function will manage connections and broadcast messages to connected clients.

The key thing is to setup [onHandshake Request] operation in APIM to invoke the Azure Function. This is the operation that will be invoked when a client initiates a WebSocket connection. The following [link](https://learn.microsoft.com/en-us/azure/api-management/websocket-api?tabs=portal#onhandshake-operation) explains how to configure the onHandshake Request operation.

We define that in our Function App as follows:

~~~csharp
using Microsoft.AspNetCore.Http;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.Azure.WebJobs.Extensions.SignalRService;
using Microsoft.Extensions.Logging;

public static class SignalRFunction
{
    [FunctionName("negotiate")]
    public static SignalRConnectionInfo GetSignalRInfo(
        [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
        [SignalRConnectionInfo(HubName = "chat")] SignalRConnectionInfo connectionInfo,
        ILogger log)
    {
        log.LogInformation("SignalR Negotiate function triggered.");
        return connectionInfo;
    }
}
~~~

This is a basic function that handles the SignalR negotiation. The function is triggered by an HTTP POST request and returns a SignalR connection info object, which contains a URL to the SignalR service and an access token. This allows clients to then connect to the SignalR service.

### Function App Deployment

~~~javascript
param location string
param resourceNamePrefix string

resource functionApp 'Microsoft.Web/sites@2021-06-01' = {
  name: '${resourceNamePrefix}-function-app'
  location: location
  kind: 'functionApp'
  properties: {
    siteConfig: {
      appSettings: [
        {
          name: 'AzureSignalRConnectionString'
          value: 'YOUR_SIGNALR_CONNECTION_STRING'
        }
      ]
    }
  }
}

output functionName string = functionApp.name
~~~