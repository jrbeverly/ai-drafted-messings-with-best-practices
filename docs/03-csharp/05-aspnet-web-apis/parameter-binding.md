# Parameter Binding

Explicit parameter binding from HTTP requests to handler method parameters using attributes.

## Why It Matters

- Explicit attributes make binding sources unambiguous
- Framework handles type conversion and validation automatically
- Value objects with `TryParse` bind seamlessly from routes and query strings

## Binding Sources

```csharp
public static async Task<IResult> HandleAsync(
    [FromRoute]  string libraryId,                          // URL path
    [FromQuery]  int page = 1,                              // query string
    [FromBody]   Request request,                           // JSON body
    [FromHeader(Name = "X-Correlation-ID")] string? corId,  // HTTP header
    ClaimsPrincipal user,                                   // auto-injected
    HttpContext context,                                    // auto-injected
    CancellationToken ct,                                   // auto-injected
    ILoanService loanService)                               // DI container
{ }
```

## Auto-Injected (no attribute needed)

`HttpContext`, `HttpRequest`, `HttpResponse`, `ClaimsPrincipal`, `CancellationToken`, and any registered DI service.

## Type Conversion

The framework converts route/query strings automatically:

| Target | Example |
|--------|---------|
| `Guid` | `/{id:guid}` |
| `int` | `?page=5` |
| `bool` | `?active=true` |
| `DateTime` | `?date=2026-02-12` |
| Value object | via `TryParse` |

## Optional Parameters

Use nullable types or default values:

```csharp
[FromQuery] string? search = null,
[FromQuery] int page = 1,
[FromQuery] bool includeInactive = false
```

## Value Object Binding

Types with a `static bool TryParse(string?, out T?)` method bind automatically. Returns 400 if parsing fails.

```csharp
[FromRoute] Email email   // calls Email.TryParse
[FromRoute] LibraryId id  // calls LibraryId.TryParse
```

## Key Recommendations

- Always use `[FromRoute]`, `[FromBody]`, `[FromQuery]` explicitly -- no implicit binding
- Parameter names must match route template names
- Prefer a `[FromBody]` object over many query parameters for complex input
- Use route parameters for resource identifiers only; query params for filtering/pagination

## Pitfalls to Avoid

- Omitting binding attributes (makes source ambiguous for readers and AI)
- Mismatched parameter name vs. route template name
- Using `[FromBody]` on GET requests (not supported)
- Too many query parameters -- consolidate into a request object

## Related

- [value-objects.md](./value-objects.md) -- TryParse binding
- [request-response-pattern.md](./request-response-pattern.md) -- request DTOs
