# Minimal API Routing

Route organization using `MapGroup` for hierarchical grouping with shared prefixes, middleware, and metadata.

## Why It Matters

- Shared configuration applied once at the group level, not repeated per endpoint
- Hierarchical groups produce clean URL structures and middleware pipelines
- `Registration.Map()` keeps endpoint files self-contained while Program.cs stays minimal

## MapGroup Hierarchy

```csharp
var api = app.MapGroup("/api")
    .AddEndpointFilter<DomainExceptionFilter>();

var v1 = api.MapGroup("/v1");

var loanGroup = v1.MapGroup("/loans/{libraryId}")
    .RequireAuthorization()
    .WithTags("Loans");

// Register endpoints
LoanCreateRoute.Registration.Map(loanGroup);
LoanGetRoute.Registration.Map(loanGroup);
```

Produces paths like `/api/v1/loans/{libraryId}/create`.

## Registration.Map() Pattern

Each endpoint exposes a static method that takes a `RouteGroupBuilder`:

```csharp
public static class Registration
{
    public static RouteHandlerBuilder Map(RouteGroupBuilder group)
        => group.MapPost("/create", Handler.HandleAsync)
            .WithName("CreateLoan")
            .WithApiVersioning(ResourceName, Version);
}
```

## Optional: Batch Extension Method

```csharp
public static class LoanEndpoints
{
    public static RouteGroupBuilder MapLoanEndpoints(this RouteGroupBuilder group)
    {
        LoanCreateRoute.Registration.Map(group);
        LoanGetRoute.Registration.Map(group);
        return group;
    }
}

v1.MapGroup("/loans/{libraryId}").MapLoanEndpoints();
```

## Shared Configuration at Group Level

- `.RequireAuthorization()` -- all child endpoints require auth
- `.WithTags("Loans")` -- all tagged for OpenAPI grouping
- `.AddEndpointFilter<T>()` -- shared filters (logging, exception handling)
- Route parameters in the group path (e.g., `{libraryId}`) are available to all child handlers

## Pitfalls to Avoid

- Putting endpoint-level logic in Program.cs instead of in `Registration.Map()`
- Forgetting that group-level middleware applies to **all** children
- Duplicating `.RequireAuthorization()` on every endpoint when it belongs on the group

## Related

- [endpoint-organization.md](./endpoint-organization.md) -- endpoint file structure
- [endpoint-filters.md](./endpoint-filters.md) -- group-level filters
