# Action Filters and Endpoint Filters

Cross-cutting concerns (logging, validation, auth, caching) applied before/after endpoint execution via reusable filter components.

## Why It Matters

- Separates cross-cutting logic from business logic, keeping endpoints clean
- Reusable across multiple endpoints without code duplication
- Supports short-circuiting to reject requests early (validation, auth)

## Key Patterns

**Minimal API -- IEndpointFilter:**

```csharp
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var request = context.Arguments.OfType<T>().FirstOrDefault();
        if (request is null) return await next(context);

        var result = await _validator.ValidateAsync(request);
        return result.IsValid ? await next(context) : Results.ValidationProblem(result.ToDictionary());
    }
}

// Apply: .AddEndpointFilter<ValidationFilter<CreateUserRequest>>()
// Chain: .AddEndpointFilter<LoggingFilter>().AddEndpointFilter<RateLimitFilter>()
// Global via group: app.MapGroup("/api").AddEndpointFilter<LoggingFilter>()
```

**Controller -- IActionFilter / IAsyncActionFilter / IExceptionFilter:**

```csharp
// Apply: [ServiceFilter(typeof(LoggingActionFilter))]
// Global: options.Filters.Add<ApiExceptionFilter>();
// Order: [ServiceFilter(typeof(TimingFilter), Order = 1)]
```

**Controller filter types:** Authorization > Resource > Action > Exception > Result

## Pitfalls to Avoid

- Putting business logic in filters (belongs in services)
- Expensive operations in filters (they run on every request)
- Forgetting to call `next()` when not intentionally short-circuiting
- Using database calls inside filters -- keep them lightweight
- Mixing endpoint filters and action filters on same resource
