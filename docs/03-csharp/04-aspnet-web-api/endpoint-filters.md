# Endpoint Filters

Endpoint filters intercept requests and responses in Minimal APIs. Add cross-cutting concerns without cluttering endpoint logic.

## Principle

Filters run before and after endpoint execution. Chain multiple filters. Keep filters focused and reusable.

## Basic Filter

```csharp
app.MapGet("/users/{id}", async (
    string id,
    IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);
    return Results.Ok(user);
})
.AddEndpointFilter(async (context, next) =>
{
    // Before endpoint execution
    Console.WriteLine("Before endpoint");

    var result = await next(context);

    // After endpoint execution
    Console.WriteLine("After endpoint");

    return result;
});
```

## Logging Filter

```csharp
public class LoggingFilter : IEndpointFilter
{
    private readonly ILogger<LoggingFilter> _logger;

    public LoggingFilter(ILogger<LoggingFilter> logger)
    {
        _logger = logger;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var method = context.HttpContext.Request.Method;
        var path = context.HttpContext.Request.Path;

        _logger.LogInformation("Request: {Method} {Path}", method, path);

        var stopwatch = Stopwatch.StartNew();
        var result = await next(context);
        stopwatch.Stop();

        _logger.LogInformation(
            "Response: {Method} {Path} - {Duration}ms",
            method, path, stopwatch.ElapsedMilliseconds);

        return result;
    }
}

// Apply to endpoints
app.MapGet("/users", GetUsers)
    .AddEndpointFilter<LoggingFilter>();
```

## Validation Filter

```csharp
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    private readonly IValidator<T> _validator;

    public ValidationFilter(IValidator<T> validator)
    {
        _validator = validator;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        // Find the parameter of type T
        var argument = context.Arguments
            .OfType<T>()
            .FirstOrDefault();

        if (argument is null)
        {
            return await next(context);
        }

        // Validate
        var validationResult = await _validator.ValidateAsync(argument);

        if (!validationResult.IsValid)
        {
            return Results.ValidationProblem(
                validationResult.ToDictionary());
        }

        return await next(context);
    }
}

// Apply to endpoint
app.MapPost("/users", async (
    CreateUserRequest request,
    IUserService userService) =>
{
    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
})
.AddEndpointFilter<ValidationFilter<CreateUserRequest>>();
```

## Authorization Filter

```csharp
public class RequireRoleFilter : IEndpointFilter
{
    private readonly string[] _roles;

    public RequireRoleFilter(params string[] roles)
    {
        _roles = roles;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var user = context.HttpContext.User;

        if (!user.Identity?.IsAuthenticated ?? true)
        {
            return Results.Unauthorized();
        }

        if (_roles.Length > 0 && !_roles.Any(role => user.IsInRole(role)))
        {
            return Results.Forbid();
        }

        return await next(context);
    }
}

// Apply with specific roles
app.MapGet("/admin/users", GetAllUsers)
    .AddEndpointFilter(new RequireRoleFilter("Admin"));
```

## Caching Filter

```csharp
public class CachingFilter : IEndpointFilter
{
    private readonly IMemoryCache _cache;

    public CachingFilter(IMemoryCache cache)
    {
        _cache = cache;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var request = context.HttpContext.Request;

        // Only cache GET requests
        if (request.Method != HttpMethods.Get)
        {
            return await next(context);
        }

        var cacheKey = $"{request.Path}{request.QueryString}";

        // Try get from cache
        if (_cache.TryGetValue(cacheKey, out var cachedResult))
        {
            return cachedResult;
        }

        // Execute endpoint
        var result = await next(context);

        // Cache for 5 minutes
        _cache.Set(cacheKey, result, TimeSpan.FromMinutes(5));

        return result;
    }
}
```

## Rate Limiting Filter

```csharp
public class RateLimitFilter : IEndpointFilter
{
    private readonly IMemoryCache _cache;
    private readonly int _maxRequests;
    private readonly TimeSpan _timeWindow;

    public RateLimitFilter(
        IMemoryCache cache,
        int maxRequests = 100,
        TimeSpan? timeWindow = null)
    {
        _cache = cache;
        _maxRequests = maxRequests;
        _timeWindow = timeWindow ?? TimeSpan.FromMinutes(1);
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var ip = context.HttpContext.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        var key = $"ratelimit:{ip}";

        var count = _cache.GetOrCreate(key, entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = _timeWindow;
            return 0;
        });

        if (count >= _maxRequests)
        {
            return Results.StatusCode(429); // Too Many Requests
        }

        _cache.Set(key, count + 1, _timeWindow);

        return await next(context);
    }
}
```

## Multiple Filters

```csharp
app.MapPost("/users", CreateUser)
    .AddEndpointFilter<LoggingFilter>()           // 1. Log request
    .AddEndpointFilter<RateLimitFilter>()         // 2. Check rate limit
    .AddEndpointFilter<ValidationFilter<CreateUserRequest>>()  // 3. Validate
    .AddEndpointFilter(async (context, next) =>   // 4. Timing
    {
        var stopwatch = Stopwatch.StartNew();
        var result = await next(context);
        stopwatch.Stop();

        context.HttpContext.Response.Headers.Add(
            "X-Response-Time",
            stopwatch.ElapsedMilliseconds.ToString());

        return result;
    });

// Execution order:
// Request → Logging → Rate Limit → Validation → Timing → Endpoint → Timing → Validation → Rate Limit → Logging → Response
```

## Filter Factory

```csharp
public static class EndpointFilterExtensions
{
    public static RouteHandlerBuilder AddValidation<T>(
        this RouteHandlerBuilder builder) where T : class
    {
        return builder.AddEndpointFilter<ValidationFilter<T>>();
    }

    public static RouteHandlerBuilder AddCaching(
        this RouteHandlerBuilder builder,
        TimeSpan? duration = null)
    {
        return builder.AddEndpointFilter(new CachingFilter(
            builder.ServiceProvider.GetRequiredService<IMemoryCache>(),
            duration ?? TimeSpan.FromMinutes(5)));
    }

    public static RouteHandlerBuilder AddRateLimit(
        this RouteHandlerBuilder builder,
        int maxRequests = 100)
    {
        return builder.AddEndpointFilter(new RateLimitFilter(
            builder.ServiceProvider.GetRequiredService<IMemoryCache>(),
            maxRequests));
    }
}

// Usage
app.MapGet("/users", GetUsers)
    .AddCaching(TimeSpan.FromMinutes(10))
    .AddRateLimit(200);
```

## Global Filters

```csharp
// Apply to all endpoints in a group
var api = app.MapGroup("/api")
    .AddEndpointFilter<LoggingFilter>()
    .AddEndpointFilter<RateLimitFilter>();

api.MapGet("/users", GetUsers);
api.MapPost("/users", CreateUser);
api.MapGet("/loans", GetLoans);

// All endpoints in /api group have logging and rate limiting
```

## Short-Circuiting

```csharp
public class AuthenticationFilter : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var token = context.HttpContext.Request.Headers["Authorization"].ToString();

        if (string.IsNullOrEmpty(token))
        {
            // Short-circuit: don't call next()
            return Results.Unauthorized();
        }

        if (!IsValidToken(token))
        {
            // Short-circuit: don't call next()
            return Results.Problem(
                type: "https://api.example.com/errors/invalid-token",
                title: "Invalid Token",
                status: 401,
                detail: "The provided token is invalid or expired");
        }

        // Token valid, proceed
        return await next(context);
    }

    private bool IsValidToken(string token) => true; // Implementation
}
```

## Accessing Context

```csharp
public class ContextExampleFilter : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        // Access HTTP context
        var httpContext = context.HttpContext;
        var request = httpContext.Request;
        var response = httpContext.Response;
        var user = httpContext.User;

        // Access endpoint arguments
        var arguments = context.Arguments;
        var firstArg = arguments.FirstOrDefault();

        // Modify arguments (if needed)
        // arguments[0] = modifiedValue;

        // Execute next filter or endpoint
        var result = await next(context);

        // Modify result (if needed)
        if (result is IResult resultObject)
        {
            // Wrap or transform result
        }

        return result;
    }
}
```

## Guidelines

**When to Use:**
- Cross-cutting concerns (logging, auth, caching)
- Request/response transformation
- Validation before endpoint execution
- Short-circuiting (early return without executing endpoint)

**Best Practices:**
- Keep filters focused (single responsibility)
- Make filters reusable
- Register dependencies in DI container
- Order filters intentionally
- Use global filters for common concerns

**Performance:**
- Filters run on every request
- Avoid expensive operations in filters
- Use caching for repeated checks
- Consider async operations

## Benefits

Reusable. Apply to multiple endpoints.

Composable. Chain multiple filters.

Testable. Easy to unit test.

Clean. Keep endpoint logic focused.

## Related

- [minimal-api-basics.md](./minimal-api-basics.md) - Endpoint definition
- [request-validation.md](./request-validation.md) - Input validation
- [middleware-pipeline.md](./middleware-pipeline.md) - Application-level middleware
- [custom-middleware.md](./custom-middleware.md) - Building middleware
