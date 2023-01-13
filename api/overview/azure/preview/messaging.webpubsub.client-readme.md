---
title: Azure Web PubSub client library for .NET
keywords: Azure, dotnet, SDK, API, Azure.Messaging.WebPubSub.Client, webpubsub
author: vicancy
ms.author: lianwei
ms.date: 01/11/2023
ms.topic: reference
ms.devlang: dotnet
ms.service: webpubsub
---
# Azure Web PubSub client library for .NET - version 1.0.0-beta.1 


[Azure Web PubSub Service](https://aka.ms/awps/doc) is an Azure-managed service that helps developers easily build web applications with real-time features and publish-subscribe patterns. Any scenario that requires real-time publish-subscribe messaging between server and clients or among clients can use Azure Web PubSub service. Traditional real-time features that often require polling from the server or submitting HTTP requests can also use Azure Web PubSub service.

You can use this library on your client side to manage the WebSocket client connections, as shown in the below diagram:

![overflow](./samples/media/overview.png)

Use this library to:

- Send messages to groups
- Send event to event handlers
- Join and leave groups
- Listen messages from groups and servers

Details about the terms used here are described in [Key concepts](#key-concepts) section.

[Source code](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Messaging.WebPubSub.Client_1.0.0-beta.1/sdk/webpubsub/Azure.Messaging.WebPubSub.Client/src) |
[API reference documentation](https://aka.ms/awps/sdk/csharp) |
[Product documentation](https://aka.ms/awps/doc) |

## Getting started

### Install the package

Install the client library from [NuGet](https://www.nuget.org/):

```dotnetcli
dotnet add package Azure.Messaging.WebPubSub.Client --prerelease
```

### Prerequisites

- An [Azure subscription][azure_sub].
- An existing Azure Web PubSub service instance. [Create Azure Web PubSub service instance](https://learn.microsoft.com/azure/azure-web-pubsub/howto-develop-create-instance)

### Authenticate the client

A Client uses a Client Access URL to connect and authenticate with the service. The Uri follow the patten as `wss://<service_name>.webpubsub.azure.com/client/hubs/<hub_name>?access_token=<token>`. There're multiple ways to get a Client Access URL. As a quick start, you can copy and paste from Azure Portal, and for production, you usually need a negotiation server to generate the Uri.

#### Use Client Access URL from Azure Portal

As a quick start, you can go to the Portal and copy the **Client Access URL** from **Key** blade.

![get_client_url](https://learn.microsoft.com/azure/azure-web-pubsub/media/howto-websocket-connect/generate-client-url.png)

As shown in the diagram, the client will be granted permission of sending messages to the specific group and joining the specific group. Learn more about client permission, see [permissions](https://learn.microsoft.com/azure/azure-web-pubsub/reference-json-reliable-webpubsub-subprotocol#permissions)

```C# Snippet:WebPubSubClient_Construct
var client = new WebPubSubClient(new Uri("<client-access-uri>"));
```

#### Use negotiation server to generate Client Access URL

In production, a client usually fetches the Client Access URL from a negotiation server. The server holds the connection string and generates the Client Access URL through `WebPubSubServiceClient`. As a sample, the code snippet below just demonstrates how to generate the Client Access URL inside a single process.

```C# Snippet:WebPubSubClient_Construct2
var client = new WebPubSubClient(new WebPubSubClientCredential(token =>
{
    // In common practice, you will have a negotiation server for generating token. Client should fetch token from it.
    return FetchClientAccessTokenFromServerAsync(token);
}));
```

```C# Snippet:WebPubSubClient_GenerateClientAccessUri
public async ValueTask<Uri> FetchClientAccessTokenFromServerAsync(CancellationToken token)
{
    var serviceClient = new WebPubSubServiceClient("<< Connection String >>", "hub");
    return await serviceClient.GetClientAccessUriAsync();
}
```

Features to differentiate `WebPubSubClient` and `WebPubSubServiceClient`.

|Class Name|WebPubSubClient|WebPubSubServiceClient|
|------|---------|---------|
|Nuget Package Name|Azure.Messaging.WebPubSub.Client |Azure.Messaging.WebPubSub|
|Features|Usually used on client side. Publish messages and subscribe to messages.|Usually used on server side. Generate Client Access Uri and manage clients|

Find more details in [Azure.Messaging.WebPubSub](https://github.com/Azure/azure-sdk-for-net/tree/Azure.Messaging.WebPubSub.Client_1.0.0-beta.1/sdk/webpubsub/Azure.Messaging.WebPubSub)

## Key concepts

### Connection

A connection, also known as a client connection, represents an individual WebSocket connection connected to the Web PubSub service. When successfully connected, the Web PubSub service assigns the connection a unique connection ID. Each `WebPubSubClient` creates its own exclusive connection.

### Recovery

If a client using reliable protocols disconnects, a new WebSocket tries to establish using the connection ID of the lost connection. If the new WebSocket connection is successfully connected, the connection is recovered. Throughout the time a client is disconnected, the service retains the client's context as well as all messages that the client was subscribed to, and when the client recovers, the service will send these messages to the client. If the service returns WebSocket error code `1008` or the recovery attempt lasts more than 30 seconds, the recovery fails.

### Reconnect

Reconnection happens when the client connection drops and fails to recover. Reconnection starts a new connection and the new connection has a new connection ID. Unlike recovery, the service treats the reconnected client as a new client connection. The client connection needs to rejoin groups. By default, the client library rejoins groups after reconnection.

### Hub

A hub is a logical concept representing a collection of client connections. Usually, you use one hub for one purpose: for example, a chat hub, or a notification hub. When a client connection is created, it connects to a hub, and during its lifetime, it is bound to that hub. Different applications can share one Azure Web PubSub service by using different hub names.

### Group

A group is a subset of connections to the hub. You can add and remove connections from a group at any time. For example, a chat room can be considered a group.  When clients join and leave the room, they are added and removed from the group. A connection can belong to multiple groups, and a group can contain multiple connections.

### User

Connections to Web PubSub can belong to one user. A user might have multiple connections, for example when a single user is connected across multiple devices or browser tabs.

## Client Lifetime

Each of the Web PubSub clients is safe to cache and use as a singleton for the lifetime of the application. The registered event callbacks share the same lifetime with the client. This means you can add or remove callbacks at any time and the registration status won't change after reconnection or even stopping the client.

## Examples

### Specify subprotocol

You can specify the subprotocol to be used by the client. By default, the client uses `json.reliable.webpubsub.azure.v1`. You can choose to use `json.reliable.webpubsub.azure.v1` or `json.webpubsub.azure.v1` as shown below.

```C# Snippet:WebPubSubClient_JsonProtocol
var client = new WebPubSubClient(new Uri("<client-access-uri>"), new WebPubSubClientOptions
{
    Protocol = new WebPubSubJsonProtocol()
});
```

```C# Snippet:WebPubSubClient_JsonReliableProtocol
var client = new WebPubSubClient(new Uri("<client-access-uri>"), new WebPubSubClientOptions
{
    Protocol = new WebPubSubJsonReliableProtocol()
});
```

### Consume messages from the server and groups

A client can add callbacks to consume messages from the server and groups. Please note, clients can only receive group messages that it has joined.

```C# Snippet:WebPubSubClient_Subscribe_ServerMessage
client.ServerMessageReceived += eventArgs =>
{
    Console.WriteLine($"Receive message: {eventArgs.Message.Data}");
    return Task.CompletedTask;
};
```

```C# Snippet:WebPubSubClient_Subscribe_GroupMessage
client.GroupMessageReceived += eventArgs =>
{
    Console.WriteLine($"Receive group message from {eventArgs.Message.Group}: {eventArgs.Message.Data}");
    return Task.CompletedTask;
};
```

### Add callbacks for connected, disconnected, and stopped events

When a client connection is connected to the service, the `Connected` event is triggered once it received the connected message from the service.

```C# Snippet:WebPubSubClient_Subscribe_Connected
client.Connected += eventArgs =>
{
    Console.WriteLine($"Connection {eventArgs.ConnectionId} is connected");
    return Task.CompletedTask;
};
```

When a client connection is disconnected and fails to recover, the `Disconnected` event is triggered.

```C# Snippet:WebPubSubClient_Subscribe_Disconnected
client.Disconnected += eventArgs =>
{
    Console.WriteLine($"Connection is disconnected");
    return Task.CompletedTask;
};
```

When a client is stopped, which means the client connection is disconnected and the client stops trying to reconnect, the `Stopped` event will be triggered. This usually happens after the `client.StopAsync()` is called, or disabled `AutoReconnect`. If you want to restart the client, you can call `client.StartAsync()` in the `Stopped` event.

```C# Snippet:WebPubSubClient_Subscribe_Stopped
client.Stopped += eventArgs =>
{
    Console.WriteLine($"Client is stopped");
    return Task.CompletedTask;
};
```

### Auto rejoin groups and handle rejoin failure

When a client connection has dropped and fails to recover, all group contexts will be cleaned up on the service side. That means when the client reconnects, it needs to rejoin groups. By default, the client enabled `AutoRejoinGroups` options. However, this feature has limitations. The client can only rejoin groups that it's originally joined by the client rather than joined by the server side. And rejoin group operations may fail due to various reasons, e.g. the client doesn't have permission to join groups. In such cases, users need to add a callback to handle the failure.

```C# Snippet:WebPubSubClient_Subscribe_RestoreFailed
client.RejoinGroupFailed += eventArgs =>
{
    Console.WriteLine($"Restore group failed");
    return Task.CompletedTask;
};
```

### Operation and retry

By default, the operation such as `client.JoinGroupAsync()`, `client.LeaveGroupAsync()`, `client.SendToGroupAsync()`, `client.SendEventAsync()` has three reties. You can use `WebPubSubClientOptions.MessageRetryOptions` to change. If all retries have failed, an error will be thrown. You can keep retrying by passing in the same `ackId` as previous retries, thus the service can help to deduplicate the operation with the same `ackId`

```C# Snippet:WebPubSubClient_JoinGroupAndRetry
// Send message to group "testGroup"
try
{
    await client.JoinGroupAsync("testGroup");
}
catch (SendMessageFailedException ex)
{
    if (ex.AckId == null)
    {
        await client.JoinGroupAsync("testGroup", ackId: ex.AckId);
    }
}
```

## Troubleshooting

### Setting up console logging

You can also [enable console logging](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Messaging.WebPubSub.Client_1.0.0-beta.1/sdk/core/Azure.Core/samples/Diagnostics.md#logging) if you want to dig deeper into the requests you're making against the service.

## Next steps

You can also find [more samples here][awps_sample].

## Contributing

This project welcomes contributions and suggestions.
Most contributions require you to agree to a Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us the rights to use your contribution.
For details, visit <https://cla.microsoft.com.>

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide a CLA and decorate the PR appropriately (e.g., label, comment).
Simply follow the instructions provided by the bot.
You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

![Impressions](https://azure-sdk-impressions.azurewebsites.net/api/impressions/azure-sdk-for-net%2Fsdk%2Ftemplate%2FAzure.Template%2FREADME.png)

[azure_sub]: https://azure.microsoft.com/free/dotnet/
[awps_sample]: https://github.com/Azure/azure-webpubsub/tree/main/samples/csharp
