# Endpoint Filters

`IEndpointFilter` provides a before/after pipeline for cross-cutting concerns like validation, logging, and authorization.

## Why It Matters

- Separates cross-cutting logic from business logic
- Reusable across endpoints; composable via registration order
- Group-level filters eliminate per-endpoint duplication

## Built-in Validation (.NET 10+)

```csharp
builder.Services.AddValidation(); // auto-validates DataAnnotations, returns 400
```

## Custom Filter Pattern

```csharp
public class LoggingFilter(ILogger<LoggingFilter> logger) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        logger.LogInformation("Executing: {Path}", context.HttpContext.Request.Path);
        return await next(context);
    }
}
```

## Applying Filters

```csharp
// Per-endpoint
group.MapPost("/create", Handler.HandleAsync)
    .AddEndpointFilter<LoggingFilter>();

// Per-group (all child endpoints inherit)
var loanGroup = v1.MapGroup("/loans/{libraryId}")
    .AddEndpointFilter<LoggingFilter>()
    .AddEndpointFilter<AuthorizationFilter>();
```

## Execution Order

Filters run in registration order (onion model):

```
LoggingFilter (before) -> AuthFilter (before) -> Handler -> AuthFilter (after) -> LoggingFilter (after)
```

## Short-Circuiting

Return early to skip the handler entirely:

```csharp
if (cached != null)
    return Results.Ok(cached); // handler never executes
```

## Good Use Cases

- Validation (complex business rules beyond DataAnnotations)
- Logging / metrics
- Authorization checks
- Caching, rate limiting
- Request/response enrichment

## Pitfalls to Avoid

- Putting endpoint-specific logic in a filter (belongs in the handler)
- Using filters for simple validation covered by DataAnnotations
- Heavy processing in filters (use middleware instead)
- Forgetting that filter order matters -- general before specific

## Related

- [minimal-api-routing.md](./minimal-api-routing.md) -- group-level filter registration
- [domain-exceptions.md](./domain-exceptions.md) -- exception handling via filter or middleware
