# SignalR Real-Time Communication

Real-time bidirectional communication. WebSockets, Server-Sent Events, Long Polling. Push notifications to clients.

## Principle

Server pushes to clients. Automatic transport selection. Strong typing with hubs. Scale with backplanes.

## Basic Hub

```csharp
public class ChatHub : Hub
{
    public async Task SendMessage(string user, string message)
    {
        // Broadcast to all clients
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }

    public async Task SendMessageToUser(string userId, string message)
    {
        // Send to specific user
        await Clients.User(userId).SendAsync("ReceiveMessage", message);
    }

    public async Task JoinGroup(string groupName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
        await Clients.Group(groupName).SendAsync("UserJoined", Context.UserIdentifier);
    }

    public async Task LeaveGroup(string groupName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
        await Clients.Group(groupName).SendAsync("UserLeft", Context.UserIdentifier);
    }

    public override async Task OnConnectedAsync()
    {
        await Clients.All.SendAsync("UserConnected", Context.ConnectionId);
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        await Clients.All.SendAsync("UserDisconnected", Context.ConnectionId);
        await base.OnDisconnectedAsync(exception);
    }
}

// Register
builder.Services.AddSignalR();

app.MapHub<ChatHub>("/chat");
```

## Strongly-Typed Hub

```csharp
public interface IChatClient
{
    Task ReceiveMessage(string user, string message);
    Task UserJoined(string userId);
    Task UserLeft(string userId);
}

public class ChatHub : Hub<IChatClient>
{
    public async Task SendMessage(string user, string message)
    {
        // Strongly-typed client method
        await Clients.All.ReceiveMessage(user, message);
    }

    public async Task JoinRoom(string roomName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, roomName);
        await Clients.Group(roomName).UserJoined(Context.UserIdentifier!);
    }
}
```

## Authentication

```csharp
// JWT authentication for SignalR
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        // ... JWT configuration ...

        // Allow SignalR to receive token from query string
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var accessToken = context.Request.Query["access_token"];

                var path = context.HttpContext.Request.Path;
                if (!string.IsNullOrEmpty(accessToken) &&
                    path.StartsWithSegments("/hub"))
                {
                    context.Token = accessToken;
                }

                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddSignalR();

// Require authentication
app.MapHub<ChatHub>("/chat")
    .RequireAuthorization();

// Access user info in hub
public class ChatHub : Hub
{
    public async Task SendMessage(string message)
    {
        var userId = Context.UserIdentifier;
        var userName = Context.User?.Identity?.Name;

        await Clients.All.SendAsync("ReceiveMessage", userName, message);
    }
}
```

## Groups

```csharp
public class NotificationHub : Hub
{
    private readonly ILogger<NotificationHub> _logger;

    public NotificationHub(ILogger<NotificationHub> logger)
    {
        _logger = logger;
    }

    public async Task SubscribeToTenant(string tenantId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"tenant_{tenantId}");
        _logger.LogInformation(
            "User {UserId} subscribed to tenant {TenantId}",
            Context.UserIdentifier,
            tenantId);
    }

    public async Task UnsubscribeFromTenant(string tenantId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"tenant_{tenantId}");
    }
}

// Send to group from API endpoint
app.MapPost("/notifications", async (
    CreateNotificationRequest request,
    IHubContext<NotificationHub> hubContext) =>
{
    await hubContext.Clients
        .Group($"tenant_{request.TenantId}")
        .SendAsync("ReceiveNotification", request.Message);

    return Results.Ok();
});
```

## Background Service Integration

```csharp
public class NotificationService : BackgroundService
{
    private readonly IHubContext<NotificationHub> _hubContext;
    private readonly IServiceProvider _serviceProvider;

    public NotificationService(
        IHubContext<NotificationHub> hubContext,
        IServiceProvider serviceProvider)
    {
        _hubContext = hubContext;
        _serviceProvider = serviceProvider;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(1));

        while (!stoppingToken.IsCancellationRequested &&
               await timer.WaitForNextTickAsync(stoppingToken))
        {
            await CheckForNotificationsAsync(stoppingToken);
        }
    }

    private async Task CheckForNotificationsAsync(CancellationToken cancellationToken)
    {
        using var scope = _serviceProvider.CreateScope();
        var repository = scope.ServiceProvider.GetRequiredService<INotificationRepository>();

        var notifications = await repository.GetPendingAsync(cancellationToken);

        foreach (var notification in notifications)
        {
            await _hubContext.Clients
                .User(notification.UserId)
                .SendAsync("ReceiveNotification", notification.Message, cancellationToken);

            await repository.MarkAsSentAsync(notification.Id, cancellationToken);
        }
    }
}

builder.Services.AddHostedService<NotificationService>();
```

## Client-Side (JavaScript)

```javascript
// Install: npm install @microsoft/signalr

import * as signalR from "@microsoft/signalr";

// Connect
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chat", {
        accessTokenFactory: () => getAccessToken()
    })
    .withAutomaticReconnect()
    .configureLogging(signalR.LogLevel.Information)
    .build();

// Receive messages
connection.on("ReceiveMessage", (user, message) => {
    console.log(`${user}: ${message}`);
});

// Start connection
await connection.start();
console.log("Connected to SignalR");

// Send message
await connection.invoke("SendMessage", "John", "Hello!");

// Join group
await connection.invoke("JoinRoom", "room1");

// Handle reconnection
connection.onreconnecting((error) => {
    console.log("Reconnecting...", error);
});

connection.onreconnected((connectionId) => {
    console.log("Reconnected:", connectionId);
});

// Cleanup
await connection.stop();
```

## Client-Side (C#/.NET Client)

```csharp
// For .NET clients (console apps, Blazor, etc.)
// Install: dotnet add package Microsoft.AspNetCore.SignalR.Client

var connection = new HubConnectionBuilder()
    .WithUrl("https://api.example.com/chat", options =>
    {
        options.AccessTokenProvider = async () => await GetAccessTokenAsync();
    })
    .WithAutomaticReconnect()
    .Build();

// Receive messages
connection.On<string, string>("ReceiveMessage", (user, message) =>
{
    Console.WriteLine($"{user}: {message}");
});

// Start connection
await connection.StartAsync();

// Send message
await connection.InvokeAsync("SendMessage", "John", "Hello from .NET!");

// Join group
await connection.InvokeAsync("JoinRoom", "room1");

// Stop connection
await connection.StopAsync();
await connection.DisposeAsync();
```

## Streaming

```csharp
// Server streaming
public class DataHub : Hub
{
    public async IAsyncEnumerable<int> StreamData(
        int count,
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        for (int i = 0; i < count; i++)
        {
            if (cancellationToken.IsCancellationRequested)
                yield break;

            yield return i;

            await Task.Delay(1000, cancellationToken);
        }
    }
}

// Client
connection.stream("StreamData", 10)
    .subscribe({
        next: (item) => console.log(item),
        complete: () => console.log("Stream completed"),
        error: (err) => console.error(err)
    });
```

## Redis Backplane (Scale-Out)

```csharp
// Install: dotnet add package Microsoft.AspNetCore.SignalR.StackExchangeRedis

builder.Services.AddSignalR()
    .AddStackExchangeRedis(options =>
    {
        options.Configuration.EndPoints.Add("localhost:6379");
        options.Configuration.ChannelPrefix = "MyApp.SignalR";
    });

// Now SignalR works across multiple servers
// Messages sent from Server A reach clients on Server B
```

## Azure SignalR Service

```csharp
// Install: dotnet add package Microsoft.Azure.SignalR

builder.Services.AddSignalR()
    .AddAzureSignalR(options =>
    {
        options.ConnectionString = builder.Configuration["Azure:SignalR:ConnectionString"];
    });

// Azure SignalR handles scaling automatically
// No need for Redis backplane
```

## Error Handling

```csharp
public class ChatHub : Hub
{
    private readonly ILogger<ChatHub> _logger;

    public async Task SendMessage(string message)
    {
        try
        {
            if (string.IsNullOrEmpty(message))
            {
                throw new HubException("Message cannot be empty");
            }

            await Clients.All.SendAsync("ReceiveMessage", Context.User?.Identity?.Name, message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error sending message");
            throw new HubException("Failed to send message");
        }
    }
}

// Client error handling
connection.on("ReceiveMessage", (user, message) => {
    console.log(`${user}: ${message}`);
});

try {
    await connection.invoke("SendMessage", "Hello!");
} catch (err) {
    console.error("Failed to send message:", err);
}
```

## Testing SignalR

```csharp
public class ChatHubTests
{
    [Fact]
    public async Task SendMessage_BroadcastsToAllClients()
    {
        // Arrange
        var mockClients = Substitute.For<IHubCallerClients>();
        var mockClientProxy = Substitute.For<IClientProxy>();

        mockClients.All.Returns(mockClientProxy);

        var hub = new ChatHub
        {
            Clients = mockClients
        };

        // Act
        await hub.SendMessage("John", "Hello");

        // Assert
        await mockClientProxy.Received(1)
            .SendCoreAsync("ReceiveMessage",
                Arg.Is<object[]>(args =>
                    args[0].ToString() == "John" &&
                    args[1].ToString() == "Hello"),
                Arg.Any<CancellationToken>());
    }
}
```

## Guidelines

**Connection Management:**
- Use automatic reconnection
- Handle reconnection events
- Implement heartbeat/ping for long connections
- Clean up connections on dispose

**Authentication:**
- Require authentication for sensitive hubs
- Pass JWT token in query string or header
- Validate user permissions in hub methods
- Use groups for access control

**Groups:**
- Use groups for targeted broadcasting
- Add/remove users from groups as needed
- Group names should be meaningful (tenant ID, room ID)
- Clean up empty groups

**Performance:**
- Use Redis backplane for multiple servers
- Consider Azure SignalR Service for scale
- Limit message size
- Throttle high-frequency updates
- Monitor active connections

**Error Handling:**
- Throw HubException for client errors
- Log errors server-side
- Implement client-side retry logic
- Handle connection failures gracefully

## Benefits

Real-time. Push updates to clients instantly.

Automatic. Transport fallback (WebSockets → SSE → Long Polling).

Scalable. Redis backplane or Azure SignalR.

Type-safe. Strongly-typed hubs and clients.

## Related

- [jwt-authentication.md](../05-auth/jwt-authentication.md) - Token authentication
- [background-services.md](./background-services.md) - Sending from background services
- [minimal-api-basics.md](../04-aspnet-web-api/minimal-api-basics.md) - Triggering from endpoints
- [dependency-injection.md](../03-dotnet/dependency-injection.md) - Injecting IHubContext
