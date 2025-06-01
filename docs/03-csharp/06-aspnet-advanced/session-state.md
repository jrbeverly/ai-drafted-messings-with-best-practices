# Session State

Store user-specific data. Session middleware. Distributed sessions. Redis cache.

## Principle

Sessions store data per user. In-memory for development. Distributed cache for production. Stateless preferred, sessions when necessary.

## Basic Session Setup

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add session services
builder.Services.AddDistributedMemoryCache(); // In-memory (development)
builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(20);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true; // GDPR compliance
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});

var app = builder.Build();

// Use session middleware (AFTER UseRouting, BEFORE endpoints)
app.UseSession();

app.MapGet("/", () => "Hello");

app.Run();
```

## Storing Session Data

```csharp
app.MapGet("/session/set", (HttpContext context) =>
{
    // Store string
    context.Session.SetString("UserName", "John Doe");

    // Store int
    context.Session.SetInt32("UserId", 123);

    // Store complex object (serialize to JSON)
    var user = new { Id = 123, Name = "John Doe", Email = "john@example.com" };
    context.Session.SetString("User", JsonSerializer.Serialize(user));

    return Results.Ok("Session data stored");
});

app.MapGet("/session/get", (HttpContext context) =>
{
    // Retrieve string
    var userName = context.Session.GetString("UserName");

    // Retrieve int
    var userId = context.Session.GetInt32("UserId");

    // Retrieve complex object
    var userJson = context.Session.GetString("User");
    var user = userJson is not null
        ? JsonSerializer.Deserialize<object>(userJson)
        : null;

    return Results.Ok(new
    {
        UserName = userName,
        UserId = userId,
        User = user
    });
});

app.MapPost("/session/clear", (HttpContext context) =>
{
    context.Session.Clear();
    return Results.Ok("Session cleared");
});
```

## Session Extension Methods

```csharp
public static class SessionExtensions
{
    public static void SetObject<T>(this ISession session, string key, T value)
    {
        session.SetString(key, JsonSerializer.Serialize(value));
    }

    public static T? GetObject<T>(this ISession session, string key)
    {
        var json = session.GetString(key);
        return json is null ? default : JsonSerializer.Deserialize<T>(json);
    }

    public static bool TryGetObject<T>(this ISession session, string key, out T? value)
    {
        var json = session.GetString(key);

        if (json is null)
        {
            value = default;
            return false;
        }

        value = JsonSerializer.Deserialize<T>(json);
        return true;
    }
}

// Usage
app.MapPost("/cart/add", (HttpContext context, CartItem item) =>
{
    var cart = context.Session.GetObject<List<CartItem>>("Cart") ?? new List<CartItem>();

    cart.Add(item);

    context.Session.SetObject("Cart", cart);

    return Results.Ok(new { ItemCount = cart.Count });
});

app.MapGet("/cart", (HttpContext context) =>
{
    var cart = context.Session.GetObject<List<CartItem>>("Cart") ?? new List<CartItem>();

    return Results.Ok(cart);
});
```

## Distributed Session (Redis)

```csharp
// Install: dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis

var builder = WebApplication.CreateBuilder(args);

// Use Redis for distributed sessions
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "MyApp:";
});

builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(20);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
});

var app = builder.Build();

app.UseSession();

// Sessions now stored in Redis
// Works across multiple servers (load balancing)
```

## Session Configuration

```csharp
builder.Services.AddSession(options =>
{
    // Session timeout (20 minutes of inactivity)
    options.IdleTimeout = TimeSpan.FromMinutes(20);

    // Cookie settings
    options.Cookie.Name = ".MyApp.Session";
    options.Cookie.HttpOnly = true; // Prevent XSS
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always; // HTTPS only
    options.Cookie.SameSite = SameSiteMode.Lax; // CSRF protection
    options.Cookie.IsEssential = true; // Bypass GDPR cookie consent

    // Cookie expiration (optional, defaults to session cookie)
    options.Cookie.Expiration = TimeSpan.FromMinutes(20);
});
```

## Lazy Session Loading

```csharp
// Session is lazy-loaded (only retrieved when accessed)
app.MapGet("/lazy-session", async (HttpContext context) =>
{
    // Session NOT loaded yet
    Console.WriteLine("Before accessing session");

    // This triggers session loading
    var userName = context.Session.GetString("UserName");

    // Session now loaded
    Console.WriteLine($"User: {userName}");

    return Results.Ok();
});
```

## Session Locking

```csharp
// Session is locked during request to prevent concurrent modifications
app.MapPost("/session/concurrent", async (HttpContext context) =>
{
    // Load session (acquires lock)
    await context.Session.LoadAsync();

    var counter = context.Session.GetInt32("Counter") ?? 0;
    counter++;

    await Task.Delay(1000); // Simulate work

    context.Session.SetInt32("Counter", counter);

    // Lock released at end of request
    return Results.Ok(new { Counter = counter });
});
```

## Shopping Cart Example

```csharp
public record CartItem(string ProductId, string Name, decimal Price, int Quantity);

public record Cart
{
    public List<CartItem> Items { get; init; } = new();
    public decimal Total => Items.Sum(i => i.Price * i.Quantity);
}

app.MapPost("/cart/add", (HttpContext context, CartItem item) =>
{
    var cart = context.Session.GetObject<Cart>("Cart") ?? new Cart();

    var existingItem = cart.Items.FirstOrDefault(i => i.ProductId == item.ProductId);

    if (existingItem is not null)
    {
        // Update quantity
        cart.Items.Remove(existingItem);
        cart.Items.Add(existingItem with { Quantity = existingItem.Quantity + item.Quantity });
    }
    else
    {
        // Add new item
        cart.Items.Add(item);
    }

    context.Session.SetObject("Cart", cart);

    return Results.Ok(new
    {
        ItemCount = cart.Items.Count,
        Total = cart.Total
    });
});

app.MapDelete("/cart/{productId}", (HttpContext context, string productId) =>
{
    var cart = context.Session.GetObject<Cart>("Cart");

    if (cart is null)
    {
        return Results.NotFound();
    }

    cart.Items.RemoveAll(i => i.ProductId == productId);

    context.Session.SetObject("Cart", cart);

    return Results.Ok(new
    {
        ItemCount = cart.Items.Count,
        Total = cart.Total
    });
});

app.MapGet("/cart", (HttpContext context) =>
{
    var cart = context.Session.GetObject<Cart>("Cart") ?? new Cart();
    return Results.Ok(cart);
});

app.MapPost("/cart/clear", (HttpContext context) =>
{
    context.Session.Remove("Cart");
    return Results.Ok();
});
```

## Session vs Database

```csharp
// ❌ BAD: Store too much in session
context.Session.SetObject("AllUsers", await GetAllUsersAsync()); // Don't store large data
context.Session.SetObject("Configuration", configuration); // Don't store static data

// ✅ GOOD: Store minimal user-specific data
context.Session.SetInt32("UserId", userId);
context.Session.SetString("UserRole", userRole);
context.Session.SetObject("Cart", cart); // Small, temporary data

// Database for persistent data
await _userService.SavePreferencesAsync(userId, preferences);
```

## Session Expiration

```csharp
app.MapGet("/session/check", (HttpContext context) =>
{
    var userName = context.Session.GetString("UserName");

    if (userName is null)
    {
        return Results.Unauthorized(); // Session expired or never set
    }

    // Accessing session extends expiration (rolling timeout)
    return Results.Ok(new { UserName = userName });
});
```

## Testing with Session

```csharp
public class SessionTests
{
    [Fact]
    public async Task AddToCart_StoresInSession()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        // Act
        var item = new CartItem("prod-1", "Product 1", 9.99m, 1);
        var response = await client.PostAsJsonAsync("/cart/add", item);

        // Assert
        response.EnsureSuccessStatusCode();

        // Get cart
        var cartResponse = await client.GetAsync("/cart");
        var cart = await cartResponse.Content.ReadFromJsonAsync<Cart>();

        Assert.NotNull(cart);
        Assert.Single(cart.Items);
        Assert.Equal("prod-1", cart.Items[0].ProductId);
    }
}
```

## Performance Considerations

```csharp
// Session has overhead
// - Cookie sent with every request
// - Session data retrieved from cache/database
// - Serialization/deserialization

// Prefer stateless when possible
app.MapGet("/stateless", (ClaimsPrincipal user) =>
{
    // User info from JWT claims (no session needed)
    var userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    var userName = user.FindFirst(ClaimTypes.Name)?.Value;

    return Results.Ok(new { UserId = userId, UserName = userName });
});

// Use session only when:
// - Data doesn't fit in JWT
// - Temporary data (shopping cart, wizard state)
// - Frequent updates (session count, activity)
```

## Guidelines

**When to Use Sessions:**
- Shopping carts
- Multi-step forms/wizards
- User preferences (temporary)
- Activity tracking

**When NOT to Use Sessions:**
- User authentication (use JWT/cookies)
- Static data (use cache)
- Large data (use database)
- API-only apps (stateless preferred)

**Production Setup:**
- Use distributed cache (Redis, SQL Server)
- Set reasonable timeout (10-30 minutes)
- Enable HTTPS only
- HttpOnly and SameSite cookies
- Monitor session store size

**Security:**
- Never store sensitive data (passwords, credit cards)
- Validate session data (don't trust)
- Clear session on logout
- Use secure cookies (HTTPS)

## Benefits

User-specific. Store data per user.

Temporary. Automatic expiration.

Distributed. Works across multiple servers with Redis.

Simple. Easy to use API.

## Related

- [tempdata-aspnet.md](./tempdata-aspnet.md) - TempData (similar but different)
- [cookie-authentication.md](../05-auth/cookie-authentication.md) - Cookie authentication
- [distributed-caching.md](../../05-dotnet/distributed-caching.md) - Redis caching
- [response-caching-middleware.md](./response-caching-middleware.md) - Response caching
