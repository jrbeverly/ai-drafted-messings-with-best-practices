# API Controllers vs Minimal APIs

Choose between class-based controllers and functional minimal APIs based on project size, complexity, and team preference.

## Why It Matters

- Wrong choice leads to unnecessary boilerplate (controllers for small APIs) or poor organization (minimal for large APIs)
- Both can coexist in the same application via hybrid approach
- Performance difference is marginal; organization and team familiarity matter more

## Decision Matrix

| Criteria | Minimal APIs | Controllers |
|----------|-------------|-------------|
| **Endpoints** | < 20 | 50+ |
| **Style** | Functional, concise | Class-based, structured |
| **Filters** | `IEndpointFilter` | Action/Result/Exception filters |
| **Validation** | Manual or filter | Automatic `ModelState` |
| **Best for** | Microservices, Lambda, prototypes | Large APIs, complex auth, MVC teams |

## Key Patterns

```csharp
// Minimal API -- organize with MapGroup + static classes
public static class UserEndpoints
{
    public static void MapUserEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/users");
        group.MapGet("/", GetUsers);
        group.MapGet("/{id}", GetUserById);
        group.MapPost("/", CreateUser);
    }
}

// Hybrid -- both in same app
app.MapControllers();           // Controllers for complex resources
app.MapGet("/health", () => Results.Ok("Healthy")); // Minimal for simple endpoints
```

## Pitfalls to Avoid

- Mixing styles for the same resource (pick one per resource)
- Using controllers for 2-3 endpoint microservices (unnecessary overhead)
- Using minimal APIs without `MapGroup` for 20+ endpoints (organization suffers)
- Migrating working controllers just because minimal APIs are newer
