# Real-Time and Service Calls

Pushing data to clients with SignalR, Server-Sent Events and WebSockets, and calling services with gRPC.

Author

Benedict Thekkel

Normal HTTP is request and response. Real-time features need the server to push data when something changes. ASP.NET Core includes SignalR for two-way messaging, Server-Sent Events for one-way streams, and gRPC for fast typed calls between services.

> **NOTE:**
>
> The code on this page has not been compiled or run yet. It targets .NET 11 and C# 15.

------------------------------------------------------------------------

## 1. Choose a Protocol

| Need | Use | Direction |
|----|----|----|
| Chat, collaboration, live dashboards with groups | SignalR | Two-way |
| Notifications, progress, LLM token streams | Server-Sent Events | Server to client |
| A custom binary or text protocol | Raw WebSockets | Two-way |
| Typed calls between backend services | gRPC | Request, or streams |

**Start with Server-Sent Events** when the client only listens. It is plain HTTP and works through most proxies. Use SignalR when clients also send messages or need groups.

------------------------------------------------------------------------

## 2. Server-Sent Events

**Server-Sent Events** (SSE) keep one HTTP response open and write events to it.

``` csharp
using System.Net.ServerSentEvents;
using System.Runtime.CompilerServices;

app.MapGet("/jobs/{id:int}/progress", (int id, CancellationToken ct) =>
{
    async IAsyncEnumerable<SseItem<int>> Stream(
        [EnumeratorCancellation] CancellationToken token)
    {
        for (var percent = 0; percent <= 100; percent += 10)
        {
            yield return new SseItem<int>(percent, eventType: "progress");
            await Task.Delay(500, token);
        }
    }

    return TypedResults.ServerSentEvents(Stream(ct));
});
```

The browser reads the stream with the built-in `EventSource` API:

``` typescript
const source = new EventSource("/jobs/7/progress");
source.addEventListener("progress", (event) => {
  console.log(JSON.parse(event.data));
});
```

`EventSource` cannot send an `Authorization` header. Use a cookie, or read the stream with `fetch`.

------------------------------------------------------------------------

## 3. SignalR Hubs

**SignalR** provides a **hub**, which is a class that clients call and that calls clients back. It chooses WebSockets when available and falls back to SSE or long polling.

``` csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.SignalR;

public interface IChatClient
{
    Task MessageReceived(string user, string text);
}

[Authorize]
public sealed class ChatHub : Hub<IChatClient>
{
    public Task JoinRoom(string room) =>
        Groups.AddToGroupAsync(Context.ConnectionId, room);

    public Task SendMessage(string room, string text) =>
        Clients.Group(room).MessageReceived(Context.User!.FindFirst("sub")!.Value, text);
}
```

``` csharp
builder.Services.AddSignalR();

var app = builder.Build();

app.MapHub<ChatHub>("/hubs/chat");
```

| Target                     | Code                    |
|----------------------------|-------------------------|
| All clients                | `Clients.All`           |
| The caller                 | `Clients.Caller`        |
| Everyone except the caller | `Clients.Others`        |
| A group                    | `Clients.Group("room")` |
| One user on all devices    | `Clients.User(userId)`  |

------------------------------------------------------------------------

## 4. SignalR From React

``` bash
npm install @microsoft/signalr
```

``` typescript
import { HubConnectionBuilder, LogLevel } from "@microsoft/signalr";

const connection = new HubConnectionBuilder()
  .withUrl("/hubs/chat", { accessTokenFactory: () => getAccessToken() })
  .withAutomaticReconnect()
  .configureLogging(LogLevel.Warning)
  .build();

connection.on("MessageReceived", (user: string, text: string) => {
  addMessage({ user, text });
});

await connection.start();
await connection.invoke("JoinRoom", "general");
await connection.invoke("SendMessage", "general", "Hello");
```

Groups are not restored after a reconnect. Join them again in `connection.onreconnected`.

------------------------------------------------------------------------

## 5. Send From Outside a Hub

Inject `IHubContext` into a service or an endpoint:

``` csharp
app.MapPost("/announce", async (
    string text,
    IHubContext<ChatHub, IChatClient> hub,
    CancellationToken ct) =>
{
    await hub.Clients.All.MessageReceived("system", text);
    return TypedResults.NoContent();
});
```

------------------------------------------------------------------------

## 6. Authentication for WebSockets

Browsers cannot set an `Authorization` header on a WebSocket. SignalR sends the token in the `access_token` query parameter, so the JWT handler must read it there.

``` csharp
.AddJwtBearer(options =>
{
    options.Events = new JwtBearerEvents
    {
        OnMessageReceived = context =>
        {
            var token = context.Request.Query["access_token"];
            if (!string.IsNullOrEmpty(token) &&
                context.HttpContext.Request.Path.StartsWithSegments("/hubs"))
            {
                context.Token = token;
            }

            return Task.CompletedTask;
        },
    };
});
```

Query strings can reach access logs. Keep these tokens short-lived and remove the parameter from proxy logs.

------------------------------------------------------------------------

## 7. Scale SignalR

Each connection belongs to one server. A message sent on one replica must reach clients on the others, so several replicas need a **backplane**.

``` bash
dotnet add package Microsoft.AspNetCore.SignalR.StackExchangeRedis
```

``` csharp
builder.Services.AddSignalR()
    .AddStackExchangeRedis(builder.Configuration.GetConnectionString("Redis")!);
```

Enable sticky sessions on the load balancer. The negotiate request and the connection must reach the same server, unless all clients use WebSockets and skip negotiation.

------------------------------------------------------------------------

## 8. Raw WebSockets

Use this level only when you need a custom protocol. You must then write reconnects, heartbeats and groups yourself.

``` csharp
using System.Net.WebSockets;

app.UseWebSockets();

app.Map("/ws/echo", async (HttpContext context, CancellationToken ct) =>
{
    if (!context.WebSockets.IsWebSocketRequest)
    {
        context.Response.StatusCode = StatusCodes.Status400BadRequest;
        return;
    }

    using var socket = await context.WebSockets.AcceptWebSocketAsync();
    var buffer = new byte[4096];

    while (socket.State == WebSocketState.Open)
    {
        var message = await socket.ReceiveAsync(buffer.AsMemory(), ct);
        if (message.MessageType == WebSocketMessageType.Close)
        {
            await socket.CloseAsync(WebSocketCloseStatus.NormalClosure, "bye", ct);
            break;
        }

        await socket.SendAsync(
            buffer.AsMemory(0, message.Count), message.MessageType, message.EndOfMessage, ct);
    }
});
```

------------------------------------------------------------------------

## 9. gRPC

**gRPC** is a remote procedure call system. It uses Protocol Buffers for a compact binary format, and it runs over HTTP/2. A `.proto` file is the contract, and the tools generate typed server and client code.

``` bash
dotnet new grpc -n InventoryService
```

``` protobuf
syntax = "proto3";

option csharp_namespace = "Inventory";

service Stock {
  rpc GetLevel (StockRequest) returns (StockReply);
}

message StockRequest {
  int32 item_id = 1;
}

message StockReply {
  int32 quantity = 1;
}
```

``` xml
<ItemGroup>
  <Protobuf Include="Protos/stock.proto" GrpcServices="Server" />
</ItemGroup>
```

``` csharp
using Grpc.Core;
using Inventory;

public sealed class StockService : Stock.StockBase
{
    public override Task<StockReply> GetLevel(StockRequest request, ServerCallContext context) =>
        Task.FromResult(new StockReply { Quantity = 42 });
}
```

``` csharp
builder.Services.AddGrpc();

var app = builder.Build();
app.MapGrpcService<StockService>();
```

Call the service from another project with the `Grpc.Net.ClientFactory` package:

``` csharp
builder.Services.AddGrpcClient<Stock.StockClient>(options =>
    options.Address = new Uri("https://inventory:8443"));
```

| Topic           | REST with JSON       | gRPC                   |
|-----------------|----------------------|------------------------|
| Format          | Text                 | Binary                 |
| Contract        | OpenAPI, optional    | `.proto`, required     |
| Browser support | Native               | Needs gRPC-Web         |
| Streaming       | Limited              | Built in               |
| Best fit        | Public APIs and SPAs | Internal service calls |

------------------------------------------------------------------------

## 10. Compared With Django Channels

| Django Channels                          | ASP.NET Core             |
|------------------------------------------|--------------------------|
| `AsyncWebsocketConsumer`                 | `Hub`                    |
| `channel_layer.group_add`                | `Groups.AddToGroupAsync` |
| `group_send`                             | `Clients.Group(name)`    |
| `channels_redis`                         | Redis backplane          |
| Daphne or Uvicorn                        | Kestrel                  |
| `get_channel_layer()` outside a consumer | `IHubContext`            |

------------------------------------------------------------------------

## 11. Common Mistakes

| Mistake | What goes wrong | Do this instead |
|----|----|----|
| No backplane with several replicas | Clients miss messages | Add the Redis backplane |
| No sticky sessions | SignalR negotiation fails | Enable them on the load balancer |
| Storing state in a hub field | A hub instance exists for one call only | Use a singleton store or Redis |
| Trusting the room name from the client | Users can join any room | Authorize in the hub method |
| Proxy buffering on SSE | Events arrive late, in batches | Disable buffering for that route |
| gRPC from the browser without gRPC-Web | The call fails | Use gRPC-Web or a REST gateway |

------------------------------------------------------------------------

Back to top
