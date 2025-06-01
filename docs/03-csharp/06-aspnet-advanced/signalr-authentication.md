# SignalR Authentication

Secure SignalR hubs. JWT bearer tokens. Connection-based authorization. Per-method authorization.

## Principle

Authenticate connections. Authorize hub methods. Validate on connect and per-method. Secure real-time communication.

## JWT Authentication for SignalR

```csharp
// Configure JWT for SignalR
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidateAudience = true,
            ValidAudience = builder.Configuration["Jwt:Audience"],
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SecretKey"]!))
        };

        // SignalR sends token in query string (not header)
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var accessToken = context.Request.Query["access_token"];

                // If the request is for SignalR hub
                var path = context.HttpContext.Request.Path;
                if (!string.IsNullOrEmpty(accessToken) &&
                    path.StartsWithSegments("/hubs"))
                {
                    context.Token = accessToken;
                }

                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddSignalR();

app.UseAuthentication();
app.UseAuthorization();

// Require authentication for hub
app.MapHub<ChatHub>("/hubs/chat")
    .RequireAuthorization();
```

## Hub with Authorization

```csharp
[Authorize] // Entire hub requires authentication
public class ChatHub : Hub
{
    private readonly ILogger<ChatHub> _logger;

    public ChatHub(ILogger<ChatHub> logger)
    {
        _logger = logger;
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        var userName = Context.User?.Identity?.Name;

        _logger.LogInformation(
            "User {UserId} ({UserName}) connected: {ConnectionId}",
            userId,
            userName,
            Context.ConnectionId);

        await base.OnConnectedAsync();
    }

    // Anyone authenticated can send messages
    public async Task SendMessage(string message)
    {
        var userName = Context.User?.Identity?.Name ?? "Unknown";

        await Clients.All.SendAsync("ReceiveMessage", userName, message);
    }

    // Only admins can broadcast announcements
    [Authorize(Roles = "Admin")]
    public async Task BroadcastAnnouncement(string announcement)
    {
        await Clients.All.SendAsync("ReceiveAnnouncement", announcement);
    }

    // Only users with specific permission
    [Authorize(Policy = "CanDeleteMessages")]
    public async Task DeleteMessage(string messageId)
    {
        // Delete logic...
        await Clients.All.SendAsync("MessageDeleted", messageId);
    }
}
```

## Client Authentication (JavaScript)

```javascript
// Create connection with access token
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/hubs/chat", {
        accessTokenFactory: () => {
            // Get token from storage or auth service
            return localStorage.getItem("access_token");
        }
    })
    .withAutomaticReconnect()
    .build();

// Handle authentication failures
connection.onclose(error => {
    if (error && error.statusCode === 401) {
        console.error("Unauthorized. Please log in again.");
        // Redirect to login
        window.location.href = "/login";
    }
});

// Start connection
await connection.start();
```

## Connection-Based Authorization

```csharp
// Custom authorization handler for connections
public class HubConnectionAuthorizationHandler :
    AuthorizationHandler<HubConnectionRequirement, HubInvocationContext>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        HubConnectionRequirement requirement,
        HubInvocationContext resource)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        if (userId is null)
        {
            return Task.CompletedTask; // Fail
        }

        // Check if user is banned
        if (context.User.HasClaim("status", "banned"))
        {
            return Task.CompletedTask; // Fail
        }

        // Check subscription for premium hubs
        if (requirement.RequiresPremium &&
            !context.User.HasClaim("subscription", "premium"))
        {
            return Task.CompletedTask; // Fail
        }

        context.Succeed(requirement);
        return Task.CompletedTask;
    }
}

public class HubConnectionRequirement : IAuthorizationRequirement
{
    public bool RequiresPremium { get; init; }
}

// Register
builder.Services.AddSingleton<IAuthorizationHandler, HubConnectionAuthorizationHandler>();

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("PremiumHub", policy =>
        policy.Requirements.Add(new HubConnectionRequirement
        {
            RequiresPremium = true
        }));
});

// Apply to hub
app.MapHub<PremiumChatHub>("/hubs/premium")
    .RequireAuthorization("PremiumHub");
```

## Group-Based Authorization

```csharp
public class PrivateChatHub : Hub
{
    private readonly IChatService _chatService;

    public PrivateChatHub(IChatService chatService)
    {
        _chatService = chatService;
    }

    // Join private room (checks membership)
    public async Task JoinRoom(string roomId)
    {
        var userId = Context.User!.FindFirst(ClaimTypes.NameIdentifier)!.Value;

        // Check if user is member of room
        var isMember = await _chatService.IsRoomMemberAsync(roomId, userId);

        if (!isMember)
        {
            throw new HubException("You are not a member of this room");
        }

        await Groups.AddToGroupAsync(Context.ConnectionId, roomId);

        await Clients.Group(roomId).SendAsync(
            "UserJoined",
            Context.User.Identity!.Name);
    }

    public async Task SendToRoom(string roomId, string message)
    {
        var userId = Context.User!.FindFirst(ClaimTypes.NameIdentifier)!.Value;

        // Verify user is still in room
        var isMember = await _chatService.IsRoomMemberAsync(roomId, userId);

        if (!isMember)
        {
            throw new HubException("You are not a member of this room");
        }

        await Clients.Group(roomId).SendAsync(
            "ReceiveMessage",
            Context.User.Identity!.Name,
            message);
    }

    public async Task LeaveRoom(string roomId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, roomId);

        await Clients.Group(roomId).SendAsync(
            "UserLeft",
            Context.User.Identity!.Name);
    }
}
```

## Custom Authentication Handler

```csharp
// For API key-based SignalR authentication
public class SignalRApiKeyAuthenticationHandler :
    AuthenticationHandler<AuthenticationSchemeOptions>
{
    private readonly IApiKeyService _apiKeyService;

    public SignalRApiKeyAuthenticationHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder,
        IApiKeyService apiKeyService)
        : base(options, logger, encoder)
    {
        _apiKeyService = apiKeyService;
    }

    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        // SignalR sends API key in query string
        if (!Request.Query.TryGetValue("api_key", out var apiKeyValues))
        {
            return AuthenticateResult.Fail("Missing API key");
        }

        var apiKey = apiKeyValues.ToString();

        var key = await _apiKeyService.ValidateAsync(apiKey);

        if (key is null)
        {
            return AuthenticateResult.Fail("Invalid API key");
        }

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, key.UserId),
            new Claim(ClaimTypes.Name, key.Name),
            new Claim("api_key_id", key.Id)
        };

        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);

        return AuthenticateResult.Success(ticket);
    }
}

// Register
builder.Services.AddAuthentication()
    .AddScheme<AuthenticationSchemeOptions, SignalRApiKeyAuthenticationHandler>(
        "SignalRApiKey",
        options => { });

// Use in hub
app.MapHub<DataHub>("/hubs/data")
    .RequireAuthorization(new AuthorizeAttribute
    {
        AuthenticationSchemes = "SignalRApiKey"
    });
```

## Rate Limiting per User

```csharp
public class RateLimitedChatHub : Hub
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<RateLimitedChatHub> _logger;

    public RateLimitedChatHub(
        IMemoryCache cache,
        ILogger<RateLimitedChatHub> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task SendMessage(string message)
    {
        var userId = Context.User!.FindFirst(ClaimTypes.NameIdentifier)!.Value;
        var cacheKey = $"rate_limit:chat:{userId}";

        // Check rate limit (max 10 messages per minute)
        var messageCount = _cache.GetOrCreate(cacheKey, entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(1);
            return 0;
        });

        if (messageCount >= 10)
        {
            _logger.LogWarning(
                "User {UserId} exceeded rate limit",
                userId);

            throw new HubException("Rate limit exceeded. Please slow down.");
        }

        // Increment count
        _cache.Set(cacheKey, messageCount + 1, TimeSpan.FromMinutes(1));

        // Send message
        await Clients.All.SendAsync(
            "ReceiveMessage",
            Context.User.Identity!.Name,
            message);
    }
}
```

## Testing Hub Authentication

```csharp
public class ChatHubTests
{
    [Fact]
    public async Task SendMessage_Authenticated_Succeeds()
    {
        // Arrange
        var hubContext = CreateHubContext(authenticated: true);

        var hub = new ChatHub(Substitute.For<ILogger<ChatHub>>())
        {
            Context = hubContext,
            Clients = Substitute.For<IHubCallerClients>()
        };

        // Act
        await hub.SendMessage("Hello");

        // Assert
        await hub.Clients.All.Received(1)
            .SendAsync("ReceiveMessage", Arg.Any<string>(), "Hello");
    }

    [Fact]
    public async Task BroadcastAnnouncement_NonAdmin_ThrowsUnauthorized()
    {
        // Arrange
        var hubContext = CreateHubContext(
            authenticated: true,
            roles: new[] { "User" }); // Not admin

        var hub = new ChatHub(Substitute.For<ILogger<ChatHub>>())
        {
            Context = hubContext
        };

        // Act & Assert
        // Authorization happens at method invocation
        // In real scenario, SignalR framework would throw
        await Assert.ThrowsAsync<HubException>(
            () => hub.BroadcastAnnouncement("Test"));
    }

    private HubCallerContext CreateHubContext(
        bool authenticated,
        string[]? roles = null)
    {
        var context = Substitute.For<HubCallerContext>();

        if (authenticated)
        {
            var claims = new List<Claim>
            {
                new(ClaimTypes.NameIdentifier, "user123"),
                new(ClaimTypes.Name, "Test User")
            };

            if (roles is not null)
            {
                foreach (var role in roles)
                {
                    claims.Add(new Claim(ClaimTypes.Role, role));
                }
            }

            var identity = new ClaimsIdentity(claims, "Test");
            var principal = new ClaimsPrincipal(identity);

            context.User.Returns(principal);
        }

        context.ConnectionId.Returns("connection123");

        return context;
    }
}
```

## Guidelines

**Connection Authentication:**
- Extract token from query string (SignalR limitation)
- Validate JWT on connection start
- Store user identity in Context.User
- Handle token expiration gracefully

**Hub Authorization:**
- Use [Authorize] attribute on hub or methods
- Check claims for fine-grained permissions
- Validate group membership before operations
- Implement custom authorization handlers for complex logic

**Security:**
- Always require authentication for production hubs
- Validate authorization on every method call
- Don't trust client-provided data (user IDs, group names)
- Rate limit per user to prevent abuse

**Error Handling:**
- Throw HubException for authorization failures
- Log unauthorized access attempts
- Return meaningful error messages to client
- Don't expose sensitive information in errors

## Benefits

Secure. Protects real-time communication.

Flexible. Role, policy, and custom authorization.

Auditable. Log all connection and authorization events.

Scalable. Works with distributed SignalR (Redis backplane).

## Related

- [signalr-basics.md](./signalr-basics.md) - SignalR fundamentals
- [jwt-authentication.md](../05-auth/jwt-authentication.md) - JWT tokens
- [policy-based-authorization.md](../05-auth/policy-based-authorization.md) - Authorization policies
- [claims-based-authorization.md](../05-auth/claims-based-authorization.md) - Claims
