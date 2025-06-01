# Custom Model Binders

Extend parameter binding for complex scenarios: multi-source binding, custom format parsing, and validation at binding time.

## Why It Matters

- Standard binding doesn't handle comma-separated lists, date ranges, or multi-source parameters
- Custom binders centralize parsing logic instead of duplicating it across endpoints
- Returning `null` from `BindAsync` triggers automatic 400 Bad Request

## Key Patterns

**Minimal API -- static `BindAsync` method:**

```csharp
public record PaginationParams
{
    public int Page { get; init; }
    public int PageSize { get; init; }

    public static ValueTask<PaginationParams?> BindAsync(HttpContext context, ParameterInfo parameter)
    {
        int.TryParse(context.Request.Query["page"], out var page);
        int.TryParse(context.Request.Query["pageSize"], out var pageSize);
        return ValueTask.FromResult<PaginationParams?>(new PaginationParams
        {
            Page = page > 0 ? page : 1,
            PageSize = pageSize is > 0 and <= 100 ? pageSize : 20
        });
    }
}
// Usage: app.MapGet("/users", (PaginationParams pagination, IUserService svc) => ...)
```

**Common custom binders:** `DateRange` (comma-separated dates), `IdList` (comma-separated IDs), `RequestContext` (route + header + query combined).

**Controller -- `IModelBinder`:**

```csharp
public class PaginationBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext ctx) { /* parse + ctx.Result = ModelBindingResult.Success(result) */ }
}
// Apply: [ModelBinder(typeof(PaginationBinder))] PaginationParams pagination
```

## Pitfalls to Avoid

- Throwing exceptions in binders (return `null` instead, let framework return 400)
- Database calls or expensive async operations inside binders
- Forgetting to apply sensible defaults for missing parameters
- Not testing edge cases: empty strings, out-of-range values, missing query params
