# Attribute Routing

Define routes declaratively on controllers and actions with `[Route]`, `[HttpGet]`, and related attributes.

## Why It Matters

- Routes live next to the code they serve, making them discoverable
- Type-safe parameter binding with compile-time checking
- Supports versioning, areas, and complex hierarchies

## Key Patterns

```csharp
[ApiController]
[Route("api/[controller]")]  // Token replacement: "api/users"
public class UsersController : ControllerBase
{
    [HttpGet]                        // GET api/users
    [HttpGet("{id}")]                // GET api/users/{id}
    [HttpGet("{id}", Name = "GetUser")]  // Named route for link generation
    [HttpPost]                       // POST api/users
    [HttpPut("{id}")]                // PUT api/users/{id}
    [HttpDelete("{id}")]             // DELETE api/users/{id}
    [HttpPost("{id}/activate")]      // Custom action
}
```

**Route parameters:** `{userId}/orders/{orderId}` -- multiple params. `{id}/profile?includeOrders=true` -- query via `[FromQuery]`.

**Multiple routes:** Stack `[HttpGet("{id}")]` and `[HttpGet("by-id/{id}")]` on same action.

**Route order:** Use `Order = 1` for specific routes, `Order = 2` for generic `{id}` catch-all.

**Minimal API equivalent:** `app.MapGet("/api/users/{id}", handler).WithName("GetUser")` or `app.MapGroup("/api/users")`.

## Pitfalls to Avoid

- Using verbs in route paths (`/getUser`) instead of nouns (`/users`)
- Too many route parameters (max 3-4; use query strings for optional filters)
- Forgetting `[ApiController]` attribute (disables automatic model validation)
- Ambiguous routes without explicit ordering
