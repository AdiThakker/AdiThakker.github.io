---
layout:     post
title:      SignalR Integration with APIM - Harnessing Real-time Communication (Post 2)
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
First to configure APIM for WebSocket, you need to create a new WebSocket API. This can be done through the Azure portal:
