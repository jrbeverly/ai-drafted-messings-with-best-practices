# REST API Principles

RESTful API design patterns. Resource-oriented, HTTP-compliant, predictable structure.

## Principle

Resources as nouns. HTTP verbs for actions. Stateless requests. Consistent patterns.

## Resource Naming

Use nouns, not verbs:

```
✅ GOOD
GET    /users              # List users
GET    /users/123          # Get user
POST   /users              # Create user
PUT    /users/123          # Update user
DELETE /users/123          # Delete user

GET    /users/123/loans    # Get user's loans
POST   /users/123/loans    # Create loan for user

❌ BAD
GET    /getUsers           # Verb in URL
POST   /createUser         # Verb in URL
GET    /user/123/getLoans  # Verb in URL
```

Plural resource names:

```
✅ GOOD
/users
/books
/loans

❌ BAD
/user
/book
/loan
```

## HTTP Methods

Use appropriate verbs:

```csharp
// GET - Retrieve resource(s)
[HttpGet("users")]
public async Task<IResult> GetUsers([FromQuery] int page = 1, [FromQuery] int limit = 20)
{
    var users = await _userService.GetUsersAsync(page, limit);
    return Results.Ok(users);
}

// GET - Retrieve single resource
[HttpGet("users/{id}")]
public async Task<IResult> GetUser(string id)
{
    var user = await _userService.GetUserByIdAsync(id);

    if (user == null)
        return Results.NotFound(new { error = "User not found" });

    return Results.Ok(user);
}

// POST - Create resource
[HttpPost("users")]
public async Task<IResult> CreateUser([FromBody] CreateUserRequest request)
{
    var user = await _userService.CreateUserAsync(request);

    return Results.Created($"/users/{user.Id}", user);
}

// PUT - Replace resource
[HttpPut("users/{id}")]
public async Task<IResult> UpdateUser(string id, [FromBody] UpdateUserRequest request)
{
    var user = await _userService.UpdateUserAsync(id, request);

    if (user == null)
        return Results.NotFound(new { error = "User not found" });

    return Results.Ok(user);
}

// PATCH - Partial update
[HttpPatch("users/{id}")]
public async Task<IResult> PatchUser(string id, [FromBody] JsonPatchDocument<User> patch)
{
    var user = await _userService.PatchUserAsync(id, patch);

    if (user == null)
        return Results.NotFound(new { error = "User not found" });

    return Results.Ok(user);
}

// DELETE - Remove resource
[HttpDelete("users/{id}")]
public async Task<IResult> DeleteUser(string id)
{
    var deleted = await _userService.DeleteUserAsync(id);

    if (!deleted)
        return Results.NotFound(new { error = "User not found" });

    return Results.NoContent();
}
```

## HTTP Status Codes

Return appropriate status codes:

```csharp
// 2xx Success
public static class StatusCodes
{
    // 200 OK - Successful GET, PUT, PATCH
    return Results.Ok(user);

    // 201 Created - Successful POST
    return Results.Created($"/users/{user.Id}", user);

    // 204 No Content - Successful DELETE
    return Results.NoContent();
}

// 4xx Client Errors
public static class ClientErrors
{
    // 400 Bad Request - Invalid input
    return Results.BadRequest(new { error = "Email is required" });

    // 401 Unauthorized - Not authenticated
    return Results.Unauthorized();

    // 403 Forbidden - Authenticated but not authorized
    return Results.Forbid();

    // 404 Not Found - Resource doesn't exist
    return Results.NotFound(new { error = "User not found" });

    // 409 Conflict - Resource conflict (duplicate email)
    return Results.Conflict(new { error = "Email already exists" });

    // 422 Unprocessable Entity - Validation failed
    return Results.UnprocessableEntity(new
    {
        error = "Validation failed",
        errors = new Dictionary<string, string[]>
        {
            ["email"] = new[] { "Email is invalid" }
        }
    });
}

// 5xx Server Errors
public static class ServerErrors
{
    // 500 Internal Server Error - Unexpected error
    return Results.Problem("Internal server error", statusCode: 500);

    // 503 Service Unavailable - Temporary unavailable
    return Results.Problem("Service unavailable", statusCode: 503);
}
```

## Request/Response Format

JSON by default:

```csharp
// Request DTO
public record CreateUserRequest
{
    [Required]
    [EmailAddress]
    public string Email { get; init; } = string.Empty;

    [Required]
    [MinLength(2)]
    public string Name { get; init; } = string.Empty;
}

// Response DTO
public record UserResponse
{
    public required string Id { get; init; }
    public required string Email { get; init; }
    public required string Name { get; init; }
    public required DateTime CreatedAt { get; init; }
}

// Endpoint
public static async Task<IResult> CreateUser(
    [FromBody] CreateUserRequest request,
    IUserService userService)
{
    var user = await userService.CreateUserAsync(request);

    return Results.Created($"/users/{user.Id}", new UserResponse
    {
        Id = user.Id,
        Email = user.Email,
        Name = user.Name,
        CreatedAt = user.CreatedAt
    });
}
```

## Pagination

Paginate list endpoints:

```csharp
// Request
public record GetUsersRequest
{
    [Range(1, int.MaxValue)]
    public int Page { get; init; } = 1;

    [Range(1, 100)]
    public int Limit { get; init; } = 20;
}

// Response
public record PagedResponse<T>
{
    public required T[] Items { get; init; }
    public required int Total { get; init; }
    public required int Page { get; init; }
    public required int Limit { get; init; }
    public required int TotalPages { get; init; }
}

// Endpoint
public static async Task<IResult> GetUsers(
    [AsParameters] GetUsersRequest request,
    IUserService userService)
{
    var (users, total) = await userService.GetUsersAsync(request.Page, request.Limit);

    return Results.Ok(new PagedResponse<UserResponse>
    {
        Items = users.Select(u => new UserResponse
        {
            Id = u.Id,
            Email = u.Email,
            Name = u.Name,
            CreatedAt = u.CreatedAt
        }).ToArray(),
        Total = total,
        Page = request.Page,
        Limit = request.Limit,
        TotalPages = (int)Math.Ceiling((double)total / request.Limit)
    });
}
```

Cursor-based pagination for large datasets:

```csharp
// Request
public record GetUsersCursorRequest
{
    public string? Cursor { get; init; }

    [Range(1, 100)]
    public int Limit { get; init; } = 20;
}

// Response
public record CursorPagedResponse<T>
{
    public required T[] Items { get; init; }
    public string? NextCursor { get; init; }
    public required bool HasMore { get; init; }
}

// Endpoint
public static async Task<IResult> GetUsersCursor(
    [AsParameters] GetUsersCursorRequest request,
    IUserService userService)
{
    var (users, nextCursor) = await userService.GetUsersByCursorAsync(
        request.Cursor,
        request.Limit);

    return Results.Ok(new CursorPagedResponse<UserResponse>
    {
        Items = users.Select(u => new UserResponse { /* ... */ }).ToArray(),
        NextCursor = nextCursor,
        HasMore = nextCursor != null
    });
}
```

## Filtering and Sorting

Query parameters for filtering:

```csharp
// Request
public record GetUsersFilteredRequest
{
    public string? Search { get; init; }
    public string? Status { get; init; }
    public DateTime? CreatedAfter { get; init; }
    public string? SortBy { get; init; } = "createdAt";
    public string? SortOrder { get; init; } = "desc";
    public int Page { get; init; } = 1;
    public int Limit { get; init; } = 20;
}

// Usage
// GET /users?search=john&status=active&sortBy=name&sortOrder=asc

// Endpoint
public static async Task<IResult> GetUsersFiltered(
    [AsParameters] GetUsersFilteredRequest request,
    IUserService userService)
{
    var (users, total) = await userService.GetUsersFilteredAsync(request);

    return Results.Ok(new PagedResponse<UserResponse>
    {
        Items = users.Select(MapToResponse).ToArray(),
        Total = total,
        Page = request.Page,
        Limit = request.Limit,
        TotalPages = (int)Math.Ceiling((double)total / request.Limit)
    });
}
```

## Nested Resources

Handle related resources:

```csharp
// Get user's loans
// GET /users/123/loans
public static async Task<IResult> GetUserLoans(
    string userId,
    ILoanService loanService)
{
    var loans = await loanService.GetLoansByUserIdAsync(userId);
    return Results.Ok(loans);
}

// Create loan for user
// POST /users/123/loans
public static async Task<IResult> CreateUserLoan(
    string userId,
    [FromBody] CreateLoanRequest request,
    ILoanService loanService)
{
    var loan = await loanService.CreateLoanAsync(userId, request);
    return Results.Created($"/loans/{loan.Id}", loan);
}

// Get specific loan for user
// GET /users/123/loans/456
public static async Task<IResult> GetUserLoan(
    string userId,
    string loanId,
    ILoanService loanService)
{
    var loan = await loanService.GetLoanAsync(loanId);

    if (loan == null || loan.UserId != userId)
        return Results.NotFound();

    return Results.Ok(loan);
}
```

## Idempotency

Ensure safe retries:

```csharp
// Use idempotency key for POST requests
public static async Task<IResult> CreateUser(
    [FromBody] CreateUserRequest request,
    [FromHeader(Name = "Idempotency-Key")] string? idempotencyKey,
    IUserService userService)
{
    if (string.IsNullOrEmpty(idempotencyKey))
        return Results.BadRequest(new { error = "Idempotency-Key header required" });

    // Check if request with this key was already processed
    var existingUser = await userService.GetUserByIdempotencyKeyAsync(idempotencyKey);
    if (existingUser != null)
    {
        // Return existing result (idempotent)
        return Results.Created($"/users/{existingUser.Id}", existingUser);
    }

    // Create new user
    var user = await userService.CreateUserAsync(request, idempotencyKey);
    return Results.Created($"/users/{user.Id}", user);
}
```

## Content Negotiation

Support multiple formats:

```csharp
// Accept header: application/json, application/xml
app.MapGet("/users/{id}", async (
    string id,
    HttpContext context,
    IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);

    if (user == null)
        return Results.NotFound();

    var accept = context.Request.Headers["Accept"].ToString();

    if (accept.Contains("application/xml"))
    {
        return Results.Content(SerializeToXml(user), "application/xml");
    }

    return Results.Ok(user);  // Default to JSON
});
```

## HATEOAS

Include links to related resources:

```csharp
public record UserResponseWithLinks
{
    public required string Id { get; init; }
    public required string Email { get; init; }
    public required string Name { get; init; }
    public required Links _links { get; init; }
}

public record Links
{
    public required Link Self { get; init; }
    public required Link Loans { get; init; }
}

public record Link
{
    public required string Href { get; init; }
    public string? Method { get; init; }
}

// Endpoint
public static async Task<IResult> GetUser(
    string id,
    IUserService userService)
{
    var user = await userService.GetUserByIdAsync(id);

    if (user == null)
        return Results.NotFound();

    return Results.Ok(new UserResponseWithLinks
    {
        Id = user.Id,
        Email = user.Email,
        Name = user.Name,
        _links = new Links
        {
            Self = new Link { Href = $"/users/{user.Id}" },
            Loans = new Link { Href = $"/users/{user.Id}/loans" }
        }
    });
}
```

## Rate Limiting

Protect API from abuse:

```csharp
// Middleware
public class RateLimitMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMemoryCache _cache;

    public async Task InvokeAsync(HttpContext context)
    {
        var clientId = GetClientId(context);
        var key = $"rate_limit:{clientId}";

        var count = _cache.GetOrCreate(key, entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(1);
            return 0;
        });

        if (count >= 100)  // 100 requests per minute
        {
            context.Response.StatusCode = 429;
            context.Response.Headers["Retry-After"] = "60";
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Rate limit exceeded. Try again in 60 seconds."
            });
            return;
        }

        _cache.Set(key, count + 1, TimeSpan.FromMinutes(1));
        await _next(context);
    }

    private string GetClientId(HttpContext context)
    {
        // Use API key or IP address
        return context.Request.Headers["X-API-Key"].FirstOrDefault()
            ?? context.Connection.RemoteIpAddress?.ToString()
            ?? "unknown";
    }
}
```

## Guidelines

**Resource Naming:**
- Use nouns, not verbs
- Plural resource names
- Lowercase with hyphens for multi-word (e.g., /loan-requests)
- Nested resources for relationships

**HTTP Methods:**
- GET for retrieval (safe, idempotent)
- POST for creation (not idempotent)
- PUT for full replacement (idempotent)
- PATCH for partial update (not idempotent)
- DELETE for removal (idempotent)

**Status Codes:**
- 2xx for success
- 4xx for client errors
- 5xx for server errors
- Use most specific code

**Request/Response:**
- JSON by default
- Consistent field naming (camelCase)
- Include metadata (timestamps, IDs)
- DTOs for input validation

**Pagination:**
- Offset/limit for simple pagination
- Cursor-based for large datasets
- Include total count and page info
- Default and max limits

**Filtering:**
- Query parameters for filters
- Consistent parameter names
- Document supported filters
- Validate filter values

## Benefits

Predictable. Consistent patterns across endpoints.

RESTful. Standard HTTP semantics.

Scalable. Stateless requests.

Developer-friendly. Intuitive resource structure.

## Related

- [api-versioning.md](./api-versioning.md) - Version management
- [api-error-handling.md](./api-error-handling.md) - Error patterns
- [openapi-documentation.md](./openapi-documentation.md) - API documentation
- [minimal-api.md](../03-csharp/01-core/minimal-api.md) - Implementation patterns
