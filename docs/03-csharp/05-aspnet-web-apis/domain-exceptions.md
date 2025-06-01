# Domain Exceptions

Typed domain exceptions that map to HTTP status codes, caught by middleware and returned as RFC 7807 Problem Details.

## Why It Matters

- Handlers stay clean -- no try-catch blocks; throw and let middleware convert
- Consistent error responses across every endpoint
- Type-safe: each exception carries its own status code and title

## Base + Concrete Exceptions

```csharp
public abstract class DomainException(int statusCode, string title, string message)
    : Exception(message)
{
    public int StatusCode { get; } = statusCode;
    public string Title { get; } = title;
}

public class NotFoundException(string entityType, string id)
    : DomainException(404, "Not Found", $"{entityType} with ID '{id}' was not found");

public class ForbiddenException(string message)
    : DomainException(403, "Forbidden", message);

public class BadRequestException(string message)
    : DomainException(400, "Bad Request", message);

public class UnauthorizedException(string message)
    : DomainException(401, "Unauthorized", message);

public class ConflictException(string message)
    : DomainException(409, "Conflict", message);
```

## Middleware (register early in pipeline)

```csharp
// Catches DomainException -> ProblemDetails; unhandled -> 500
app.UseMiddleware<DomainExceptionMiddleware>();
```

Key behaviors: log domain exceptions as **warnings**, unhandled as **errors**; always return `ProblemDetails` JSON.

## Alternative: Endpoint Filter

```csharp
var api = app.MapGroup("/api")
    .AddEndpointFilter<DomainExceptionFilter>();
```

## Usage in Handlers

```csharp
var userId = authPolicy.RequireUserId(user);                // throws UnauthorizedException
await libraryPolicy.RequireActiveMembershipAsync(userId, libraryId); // throws ForbiddenException
var book = await bookPolicy.RequireBookExistsAsync(libraryId, bookId); // throws NotFoundException
```

## When to Throw vs. Not

| Throw | Don't Throw |
|-------|-------------|
| Authorization failures | Simple validation (use DataAnnotations) |
| Resource not found | Expected control flow |
| Business rule violations | Performance-critical hot paths |
| Invalid state transitions | |

## Pitfalls to Avoid

- Wrapping handlers in try-catch instead of relying on middleware
- Using generic `Exception` instead of typed domain exceptions
- Forgetting to register middleware before endpoint mapping
- Logging domain exceptions as errors (they are expected -- use warnings)

## Related

- [policy-services.md](./policy-services.md) -- services that throw these exceptions
- [api-error-responses.md](./api-error-responses.md) -- Problem Details format
