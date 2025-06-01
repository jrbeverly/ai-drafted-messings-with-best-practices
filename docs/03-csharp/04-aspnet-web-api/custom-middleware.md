# Custom Middleware

Reusable request/response processing components for cross-cutting concerns like logging, auth, tenant resolution, and rate limiting.

## Why It Matters

- Middleware runs for all requests in the pipeline, not just specific endpoints
- Encapsulates cross-cutting concerns that don't belong in business logic
- Two styles: convention-based (constructor takes `RequestDelegate`) and factory-based (`IMiddleware`)

## Key Patterns

```csharp
// Convention-based middleware
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    public RequestTimingMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        try { await _next(context); }
        finally { /* log sw.ElapsedMilliseconds */ }
    }
}

// Extension method for clean registration
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseRequestTiming(this IApplicationBuilder b)
        => b.UseMiddleware<RequestTimingMiddleware>();
}

// Configurable via IOptions<T>
builder.Services.Configure<RateLimitOptions>(o => { o.Limit = 200; });
```

**Common patterns:** API key validation, tenant resolution (header/subdomain), request ID generation, rate limiting, error handling with `ProblemDetails`.

**Short-circuiting:** Return early without calling `_next(context)` for auth failures, rate limits.

**Response modification:** Use `context.Response.OnStarting()` to add headers before response is sent.

## Pitfalls to Avoid

- Forgetting to call `_next(context)` (silently swallows all requests)
- Blocking calls in middleware (use async/await throughout)
- Modifying response headers after response has started (use `OnStarting`)
- Not testing exception scenarios (middleware should log even when `_next` throws)
