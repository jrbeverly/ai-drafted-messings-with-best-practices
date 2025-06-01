# ASP.NET Core Middleware

Custom middleware for cross-cutting concerns in the HTTP request/response pipeline.

## Why It Matters

- Centralizes cross-cutting logic (logging, error handling, correlation IDs) in one place
- Pipeline ordering gives precise control over when logic executes
- Keeps endpoint handlers focused on business logic

## Convention-Based Middleware

```csharp
public class RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        await next(context);
        sw.Stop();
        if (sw.ElapsedMilliseconds > 1000)
            logger.LogWarning("Slow request: {Method} {Path} {ElapsedMs}ms",
                context.Request.Method, context.Request.Path, sw.ElapsedMilliseconds);
    }
}

app.UseMiddleware<RequestTimingMiddleware>();
```

## IMiddleware (DI-Friendly)

```csharp
public class CorrelationIdMiddleware : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var id = context.Request.Headers["X-Correlation-ID"].FirstOrDefault()
            ?? Guid.NewGuid().ToString();
        context.Response.Headers["X-Correlation-ID"] = id;
        context.Items["CorrelationId"] = id;
        await next(context);
    }
}

builder.Services.AddScoped<CorrelationIdMiddleware>();
app.UseMiddleware<CorrelationIdMiddleware>();
```

## Global Exception Handling

```csharp
public async Task InvokeAsync(HttpContext context)
{
    try { await _next(context); }
    catch (DomainException ex)
    {
        context.Response.StatusCode = ex.StatusCode;
        context.Response.ContentType = "application/problem+json";
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = ex.StatusCode, Title = ex.Title, Detail = ex.Message
        });
    }
}
```

## Pipeline Order (Critical)

```csharp
app.UseMiddleware<GlobalExceptionMiddleware>();   // 1. Exceptions first
app.UseHttpsRedirection();                        // 2. Security
app.UseRouting();                                 // 3. Routing
app.UseCors();                                    // 4. CORS
app.UseAuthentication();                          // 5. Auth
app.UseAuthorization();                           // 6. Authz
app.UseMiddleware<RequestTimingMiddleware>();      // 7. Observability
app.MapHealthChecks("/health");                   // 8. Endpoints
```

## Extension Methods for Clean Registration

```csharp
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseCorrelationId(this IApplicationBuilder app)
        => app.UseMiddleware<CorrelationIdMiddleware>();
}
```

## Pitfalls to Avoid

- Wrong pipeline order (exception middleware must be first to catch everything)
- Putting business logic in middleware (use services/handlers)
- Forgetting to call `await next(context)` (silently swallows the rest of the pipeline)
- Body logging in production (performance cost and potential PII exposure)

## Related

- [logging-ilogger.md](./logging-ilogger.md) -- Logging in middleware
- [dependency-injection.md](./dependency-injection.md) -- Middleware DI
