# Response Caching Middleware

HTTP response caching. Cache-Control headers. VaryBy parameters. Browser and proxy caching.

## Principle

Response caching middleware adds Cache-Control headers. Browser caches responses. Reduces server load.

## Basic Response Caching

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add response caching services
builder.Services.AddResponseCaching();

var app = builder.Build();

// Use response caching middleware (early in pipeline)
app.UseResponseCaching();

app.MapGet("/api/data", () =>
{
    return Results.Ok(new
    {
        Data = "Cached data",
        Timestamp = DateTime.UtcNow
    });
});

app.Run();
```

## ResponseCache Attribute

```csharp
// Apply to endpoint
app.MapGet("/api/public-data", () =>
{
    return Results.Ok(new { Data = "Public data" });
})
.WithMetadata(new ResponseCacheAttribute
{
    Duration = 60, // Cache for 60 seconds
    Location = ResponseCacheLocation.Any, // Client, proxy, or server
    VaryByQueryKeys = new[] { "page", "limit" }
});

// Equivalent headers:
// Cache-Control: public, max-age=60
// Vary: page, limit
```

## Cache Duration

```csharp
// Short cache (30 seconds)
app.MapGet("/api/trending", () => GetTrending())
    .WithMetadata(new ResponseCacheAttribute { Duration = 30 });

// Medium cache (5 minutes)
app.MapGet("/api/products", () => GetProducts())
    .WithMetadata(new ResponseCacheAttribute { Duration = 300 });

// Long cache (1 hour)
app.MapGet("/api/categories", () => GetCategories())
    .WithMetadata(new ResponseCacheAttribute { Duration = 3600 });

// Immutable (1 year)
app.MapGet("/api/static/{id}", (string id) => GetStaticData(id))
    .WithMetadata(new ResponseCacheAttribute
    {
        Duration = 31536000,
        Location = ResponseCacheLocation.Any
    });
```

## Cache Location

```csharp
// Client cache only (browser)
app.MapGet("/api/user-preferences", () => GetUserPreferences())
    .WithMetadata(new ResponseCacheAttribute
    {
        Duration = 300,
        Location = ResponseCacheLocation.Client // private
    });
// Cache-Control: private, max-age=300

// Any cache (browser, CDN, proxy)
app.MapGet("/api/public-data", () => GetPublicData())
    .WithMetadata(new ResponseCacheAttribute
    {
        Duration = 3600,
        Location = ResponseCacheLocation.Any // public
    });
// Cache-Control: public, max-age=3600

// No caching
app.MapGet("/api/sensitive-data", () => GetSensitiveData())
    .WithMetadata(new ResponseCacheAttribute
    {
        Location = ResponseCacheLocation.None,
        NoStore = true
    });
// Cache-Control: no-store, no-cache
```

## VaryBy Parameters

```csharp
// Vary by query string
app.MapGet("/api/products", (int page, int limit) => GetProducts(page, limit))
    .WithMetadata(new ResponseCacheAttribute
    {
        Duration = 60,
        VaryByQueryKeys = new[] { "page", "limit" }
    });

// Vary by header
app.MapGet("/api/data", (HttpContext context) =>
{
    var acceptLanguage = context.Request.Headers.AcceptLanguage.ToString();
    return GetLocalizedData(acceptLanguage);
})
.WithMetadata(new ResponseCacheAttribute
{
    Duration = 300,
    VaryByHeader = "Accept-Language"
});
// Cache-Control: public, max-age=300
// Vary: Accept-Language
```

## Manual Cache Headers

```csharp
app.MapGet("/api/custom-cache", (HttpContext context) =>
{
    // Set Cache-Control manually
    context.Response.Headers.CacheControl = "public, max-age=3600";

    // Vary by header
    context.Response.Headers.Vary = "Accept-Encoding";

    // ETag
    context.Response.Headers.ETag = "\"abc123\"";

    // Last-Modified
    context.Response.Headers.LastModified = DateTime.UtcNow.ToString("R");

    return Results.Ok(new { Data = "value" });
});
```

## Conditional Requests (ETag)

```csharp
app.MapGet("/api/products/{id}", (string id, HttpContext context) =>
{
    var product = GetProduct(id);

    if (product is null)
    {
        return Results.NotFound();
    }

    // Generate ETag from content hash
    var json = JsonSerializer.Serialize(product);
    var hash = Convert.ToBase64String(SHA256.HashData(Encoding.UTF8.GetBytes(json)));
    var etag = $"\"{hash}\"";

    // Check If-None-Match header
    var ifNoneMatch = context.Request.Headers.IfNoneMatch.ToString();

    if (ifNoneMatch == etag)
    {
        return Results.StatusCode(304); // Not Modified
    }

    // Add ETag to response
    context.Response.Headers.ETag = etag;
    context.Response.Headers.CacheControl = "public, max-age=60";

    return Results.Ok(product);
});
```

## Last-Modified Header

```csharp
app.MapGet("/api/posts/{id}", (string id, HttpContext context) =>
{
    var post = GetPost(id);

    if (post is null)
    {
        return Results.NotFound();
    }

    // Check If-Modified-Since header
    var ifModifiedSince = context.Request.Headers.IfModifiedSince.ToString();

    if (!string.IsNullOrEmpty(ifModifiedSince) &&
        DateTime.TryParse(ifModifiedSince, out var sinceDate))
    {
        if (post.UpdatedAt <= sinceDate)
        {
            return Results.StatusCode(304); // Not Modified
        }
    }

    // Add Last-Modified header
    context.Response.Headers.LastModified = post.UpdatedAt.ToString("R"); // RFC 1123 format
    context.Response.Headers.CacheControl = "public, max-age=300";

    return Results.Ok(post);
});
```

## No Cache for Authenticated Requests

```csharp
// Never cache authenticated responses
app.MapGet("/api/user/profile", (ClaimsPrincipal user) =>
{
    var userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value!;
    var profile = GetUserProfile(userId);

    return Results.Ok(profile);
})
.RequireAuthorization()
.WithMetadata(new ResponseCacheAttribute
{
    Location = ResponseCacheLocation.None,
    NoStore = true
});
// Cache-Control: no-store, no-cache
```

## Cache Profile Configuration

```csharp
// Define cache profiles
builder.Services.AddControllers(options =>
{
    options.CacheProfiles.Add("Default", new CacheProfile
    {
        Duration = 60,
        Location = ResponseCacheLocation.Any
    });

    options.CacheProfiles.Add("Never", new CacheProfile
    {
        Location = ResponseCacheLocation.None,
        NoStore = true
    });

    options.CacheProfiles.Add("Static", new CacheProfile
    {
        Duration = 31536000,
        Location = ResponseCacheLocation.Any
    });
});

// Use profiles
app.MapGet("/api/data", () => GetData())
    .WithMetadata(new ResponseCacheAttribute { CacheProfileName = "Default" });

app.MapGet("/api/sensitive", () => GetSensitive())
    .WithMetadata(new ResponseCacheAttribute { CacheProfileName = "Never" });
```

## Vary by User

```csharp
// Custom vary by user ID
public class VaryByUserMiddleware
{
    private readonly RequestDelegate _next;

    public VaryByUserMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        if (userId is not null)
        {
            // Add custom vary header
            context.Response.Headers.Vary = "X-User-Id";
            context.Response.Headers.Append("X-User-Id", userId);
        }

        await _next(context);
    }
}

app.UseMiddleware<VaryByUserMiddleware>();
```

## Response Caching vs Output Caching

```csharp
// Response Caching (.NET 6 and earlier)
// - Adds Cache-Control headers
// - Browser/proxy caching
// - No server-side caching

builder.Services.AddResponseCaching();
app.UseResponseCaching();

// Output Caching (.NET 7+)
// - Server-side response caching
// - Faster than response caching
// - Can use Redis, SQL Server

builder.Services.AddOutputCache();
app.UseOutputCache();

app.MapGet("/api/data", () => GetData())
    .CacheOutput(); // Server-side cache

// Use both for maximum performance:
// - Output caching for server-side
// - Response caching for client-side
```

## Bypass Cache

```csharp
app.MapGet("/api/data", (HttpContext context) =>
{
    var bypassCache = context.Request.Query.ContainsKey("nocache");

    if (bypassCache)
    {
        // Don't cache this response
        context.Response.Headers.CacheControl = "no-cache, no-store";
    }
    else
    {
        context.Response.Headers.CacheControl = "public, max-age=60";
    }

    return Results.Ok(GetData());
});

// /api/data → cached
// /api/data?nocache → not cached
```

## Testing Cache Headers

```csharp
public class CacheHeaderTests
{
    [Fact]
    public async Task GetPublicData_ReturnsCacheHeaders()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        // Act
        var response = await client.GetAsync("/api/public-data");

        // Assert
        response.EnsureSuccessStatusCode();

        Assert.True(response.Headers.CacheControl?.Public);
        Assert.Equal(60, response.Headers.CacheControl?.MaxAge?.TotalSeconds);
    }

    [Fact]
    public async Task GetSensitiveData_ReturnsNoCacheHeaders()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        // Act
        var response = await client.GetAsync("/api/sensitive-data");

        // Assert
        response.EnsureSuccessStatusCode();

        Assert.True(response.Headers.CacheControl?.NoStore);
        Assert.True(response.Headers.CacheControl?.NoCache);
    }
}
```

## CDN Integration

```csharp
app.MapGet("/api/cdn-data", (HttpContext context) =>
{
    // Cache at CDN for 1 hour
    context.Response.Headers.CacheControl = "public, max-age=3600";

    // CDN-specific headers (Cloudflare example)
    context.Response.Headers.Append("CDN-Cache-Control", "max-age=3600");

    // Surrogate control (shared caches)
    context.Response.Headers.Append("Surrogate-Control", "max-age=3600");

    return Results.Ok(GetData());
});
```

## Guidelines

**When to Cache:**
- Public, static data (categories, products)
- Slow-changing data (configuration)
- Expensive computations (reports, analytics)
- Static assets (images, CSS, JS)

**When NOT to Cache:**
- User-specific data (unless vary by user)
- Frequently changing data (live updates)
- Sensitive data (private information)
- POST/PUT/DELETE responses

**Cache Duration:**
- Frequently updated: 30-60 seconds
- Hourly updates: 5-15 minutes
- Daily updates: 1-6 hours
- Static: 1 year (31536000 seconds)

**Security:**
- Never cache authenticated responses (unless explicit)
- Use `private` for user-specific data
- Use `no-store` for sensitive data
- Validate cached data on retrieval

## Benefits

Performance. Reduces server load.

Bandwidth. Less data transfer.

Scalability. CDN and proxy caching.

User Experience. Faster page loads.

## Related

- [output-caching.md](./output-caching.md) - Server-side caching (.NET 7+)
- [response-compression.md](../04-aspnet-web-api/response-compression.md) - Compression
- [distributed-caching.md](../../05-dotnet/distributed-caching.md) - Redis caching
- [cdn-optimization.md](../../10-performance/cdn-optimization.md) - CDN best practices
