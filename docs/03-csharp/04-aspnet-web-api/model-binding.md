# Model Binding and Parameter Binding

Automatic binding of HTTP request data to method parameters. Extract data from route, query, body, headers, and services.

## Principle

Explicit binding source. Type-safe parameters. Leverage DI for services.

## Route Parameters

```csharp
// Single parameter
app.MapGet("/users/{id}", (string id) =>
    $"User ID: {id}");

// Multiple parameters
app.MapGet("/users/{userId}/loans/{loanId}", (
    string userId,
    string loanId) =>
    $"User: {userId}, Loan: {loanId}");

// Typed parameters
app.MapGet("/users/{id:int}", (int id) =>
    $"User ID: {id}");

// Optional parameters with default
app.MapGet("/users/{id}", (string id = "default") =>
    $"User ID: {id}");
```

## Query Parameters

```csharp
// Single query parameter
app.MapGet("/users", (int page) =>
    $"Page: {page}");
// GET /users?page=1

// Multiple query parameters
app.MapGet("/users", (int page, int limit) =>
    $"Page: {page}, Limit: {limit}");
// GET /users?page=1&limit=20

// Optional query parameters
app.MapGet("/users", (int? page, int? limit) =>
    $"Page: {page ?? 1}, Limit: {limit ?? 20}");

// Default values
app.MapGet("/users", (int page = 1, int limit = 20) =>
    $"Page: {page}, Limit: {limit}");
```

## Body Binding

```csharp
// From JSON body
app.MapPost("/users", async (
    CreateUserRequest request,
    IUserService userService) =>
{
    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});

// Request body:
// {
//   "email": "user@example.com",
//   "name": "John Doe"
// }

public record CreateUserRequest(
    string Email,
    string Name,
    int Age);
```

## Header Binding

```csharp
// Single header
app.MapGet("/users/me", (
    [FromHeader(Name = "Authorization")] string authorization) =>
    $"Token: {authorization}");

// Multiple headers
app.MapGet("/data", (
    [FromHeader(Name = "X-API-Key")] string apiKey,
    [FromHeader(Name = "X-Request-ID")] string requestId) =>
    $"API Key: {apiKey}, Request ID: {requestId}");

// Optional header
app.MapGet("/data", (
    [FromHeader(Name = "X-Optional")] string? optional) =>
    $"Optional header: {optional ?? "not provided"}");
```

## Service Injection

```csharp
// Inject services via DI
app.MapGet("/users", async (
    IUserService userService,
    ILogger<Program> logger) =>
{
    logger.LogInformation("Fetching all users");
    var users = await userService.GetUsersAsync();
    return Results.Ok(users);
});

// Multiple services
app.MapPost("/users", async (
    CreateUserRequest request,
    IUserService userService,
    IValidator<CreateUserRequest> validator,
    ILogger<Program> logger) =>
{
    logger.LogInformation("Creating user: {Email}", request.Email);

    var validationResult = await validator.ValidateAsync(request);
    if (!validationResult.IsValid)
    {
        return Results.ValidationProblem(validationResult.ToDictionary());
    }

    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});
```

## HttpContext Access

```csharp
// Access HttpContext
app.MapGet("/users/me", async (
    HttpContext context,
    IUserService userService) =>
{
    var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

    if (userId is null)
    {
        return Results.Unauthorized();
    }

    var user = await userService.GetUserByIdAsync(userId);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});

// Access request properties
app.MapGet("/info", (HttpRequest request) =>
{
    return Results.Ok(new
    {
        method = request.Method,
        path = request.Path,
        query = request.QueryString.ToString(),
        headers = request.Headers.Select(h => new { h.Key, h.Value })
    });
});
```

## Form Data Binding

```csharp
// From form data
app.MapPost("/upload", async (
    [FromForm] string name,
    [FromForm] IFormFile file) =>
{
    if (file.Length == 0)
    {
        return Results.BadRequest("File is required");
    }

    var path = Path.Combine("uploads", file.FileName);
    using var stream = File.OpenWrite(path);
    await file.CopyToAsync(stream);

    return Results.Ok(new { name, fileName = file.FileName });
});

// Form model
public record UploadRequest(
    [FromForm] string Name,
    [FromForm] IFormFile File);

app.MapPost("/upload", async (
    [AsParameters] UploadRequest request) =>
{
    // Process request.Name and request.File
});
```

## AsParameters

```csharp
// Group parameters into a record
public record GetUsersQuery(
    int Page = 1,
    int Limit = 20,
    string? Search = null,
    string? SortBy = null);

app.MapGet("/users", async (
    [AsParameters] GetUsersQuery query,
    IUserService userService) =>
{
    var users = await userService.GetUsersAsync(
        query.Page,
        query.Limit,
        query.Search,
        query.SortBy);

    return Results.Ok(users);
});

// With services
public record GetUserRequest(
    string Id,
    IUserService UserService,
    ILogger<Program> Logger);

app.MapGet("/users/{id}", async (
    [AsParameters] GetUserRequest request) =>
{
    request.Logger.LogInformation("Fetching user: {Id}", request.Id);
    var user = await request.UserService.GetUserByIdAsync(request.Id);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});
```

## Custom Binding

```csharp
// Bind from custom source
public record PaginationRequest
{
    public int Page { get; init; }
    public int Limit { get; init; }

    public static ValueTask<PaginationRequest?> BindAsync(
        HttpContext context,
        ParameterInfo parameter)
    {
        const int maxLimit = 100;

        int.TryParse(context.Request.Query["page"], out var page);
        int.TryParse(context.Request.Query["limit"], out var limit);

        var result = new PaginationRequest
        {
            Page = page > 0 ? page : 1,
            Limit = limit > 0 ? Math.Min(limit, maxLimit) : 20
        };

        return ValueTask.FromResult<PaginationRequest?>(result);
    }
}

// Usage
app.MapGet("/users", async (
    PaginationRequest pagination,
    IUserService userService) =>
{
    var users = await userService.GetUsersAsync(
        pagination.Page,
        pagination.Limit);

    return Results.Ok(users);
});
```

## CancellationToken

```csharp
// Automatic cancellation token binding
app.MapGet("/users", async (
    IUserService userService,
    CancellationToken cancellationToken) =>
{
    var users = await userService.GetUsersAsync(cancellationToken);
    return Results.Ok(users);
});

// Long-running operation
app.MapGet("/export", async (
    IUserService userService,
    CancellationToken cancellationToken) =>
{
    var stream = new MemoryStream();

    await userService.ExportUsersAsync(stream, cancellationToken);

    stream.Position = 0;
    return Results.File(stream, "text/csv", "users.csv");
});
```

## ClaimsPrincipal

```csharp
// Access current user
app.MapGet("/users/me", async (
    ClaimsPrincipal user,
    IUserService userService) =>
{
    var userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value;

    if (userId is null)
    {
        return Results.Unauthorized();
    }

    var currentUser = await userService.GetUserByIdAsync(userId);
    return currentUser is not null ? Results.Ok(currentUser) : Results.NotFound();
});

// Check claims
app.MapGet("/admin/users", async (
    ClaimsPrincipal user,
    IUserService userService) =>
{
    if (!user.IsInRole("Admin"))
    {
        return Results.Forbid();
    }

    var users = await userService.GetAllUsersAsync();
    return Results.Ok(users);
});
```

## Binding Precedence

Order of parameter binding:

1. **Route values** - `{id}` in route template
2. **Query string** - `?page=1&limit=20`
3. **Form values** - `[FromForm]`
4. **Body (JSON)** - Complex types without `[FromX]` attribute
5. **Services (DI)** - Registered services
6. **Special types** - HttpContext, ClaimsPrincipal, CancellationToken

```csharp
app.MapPost("/users/{id}", async (
    string id,              // From route
    int page,               // From query
    CreateUserRequest body, // From JSON body
    IUserService service,   // From DI
    HttpContext context,    // Special type
    CancellationToken ct    // Special type
    ) =>
{
    // Use parameters
});
```

## Binding Errors

```csharp
// Handle binding failures
app.MapGet("/users/{id:int}", (int id) =>
    $"User ID: {id}");
// GET /users/abc → 400 Bad Request

// Custom error handling
builder.Services.Configure<RouteHandlerOptions>(options =>
{
    options.ThrowOnBadRequest = false;
});

app.MapGet("/users", (int? id) =>
{
    if (id is null)
    {
        return Results.BadRequest(new { error = "Invalid ID format" });
    }

    return Results.Ok(new { id });
});
```

## Guidelines

**Route Parameters:**
- Use for resource identifiers (`/users/{id}`)
- Apply constraints (`{id:int}`, `{guid:guid}`)
- Keep short and semantic

**Query Parameters:**
- Use for filtering, pagination, sorting
- Optional parameters with defaults
- Validate ranges and formats

**Body Binding:**
- Use for create/update operations
- Strongly-typed DTOs
- Validate with FluentValidation

**Headers:**
- Use for metadata (auth tokens, request IDs)
- Explicit `[FromHeader]` attribute
- Handle missing headers gracefully

**Services:**
- Inject via DI
- Keep parameter lists focused
- Use `[AsParameters]` for grouping

## Benefits

Type-safe. Compile-time checking.

Explicit. Clear data sources.

Testable. Easy to mock dependencies.

Flexible. Multiple binding sources.

## Related

- [minimal-api-basics.md](./minimal-api-basics.md) - Endpoint definition
- [request-validation.md](./request-validation.md) - Input validation
- [dependency-injection.md](../03-dotnet/dependency-injection.md) - Service registration
