# Result Types in ASP.NET Core

IActionResult, ActionResult<T>, Results, TypedResults. Return types for endpoints.

## Principle

Result types define what endpoints return. Use typed results for clarity. Results class for minimal APIs. ActionResult for controllers.

## Results (Minimal APIs)

```csharp
// Results class - recommended for Minimal APIs
app.MapGet("/users/{id}", async (string id, IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);

    if (user is null)
    {
        return Results.NotFound();
    }

    return Results.Ok(user);
});

// Common Results methods
app.MapGet("/demo", () =>
{
    // 200 OK
    Results.Ok(new { Message = "Success" });

    // 201 Created
    Results.Created("/users/123", new { Id = "123", Name = "John" });

    // 202 Accepted
    Results.Accepted("/jobs/456", new { JobId = "456" });

    // 204 No Content
    Results.NoContent();

    // 400 Bad Request
    Results.BadRequest(new { Error = "Invalid input" });

    // 401 Unauthorized
    Results.Unauthorized();

    // 403 Forbidden
    Results.Forbid();

    // 404 Not Found
    Results.NotFound();

    // 409 Conflict
    Results.Conflict();

    // 500 Internal Server Error
    Results.Problem("Internal error");

    // Custom status code
    Results.StatusCode(418); // I'm a teapot
});
```

## TypedResults (Minimal APIs)

```csharp
// TypedResults - strongly-typed results
app.MapGet("/users/{id}", async (string id, IUserService userService)
    : Task<Results<Ok<User>, NotFound>> =>
{
    var user = await userService.GetUserByIdAsync(id);

    if (user is null)
    {
        return TypedResults.NotFound();
    }

    return TypedResults.Ok(user);
});

// OpenAPI benefits - knows all possible return types
// Returns:
// - 200 OK with User object
// - 404 Not Found
```

## ActionResult<T> (Controllers)

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(User), 200)]
    [ProducesResponseType(404)]
    public async Task<ActionResult<User>> GetUser(string id)
    {
        var user = await _userService.GetUserByIdAsync(id);

        if (user is null)
        {
            return NotFound();
        }

        return user; // Implicit conversion to Ok(user)
    }

    [HttpPost]
    [ProducesResponseType(typeof(User), 201)]
    [ProducesResponseType(400)]
    public async Task<ActionResult<User>> CreateUser(CreateUserRequest request)
    {
        if (!ModelState.IsValid)
        {
            return BadRequest(ModelState);
        }

        var user = await _userService.CreateUserAsync(request);

        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }
}
```

## JSON Result

```csharp
// Return JSON with custom options
app.MapGet("/custom-json", () =>
{
    var data = new { Name = "John", Age = 30 };

    var options = new JsonSerializerOptions
    {
        PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower,
        WriteIndented = true
    };

    return Results.Json(data, options, statusCode: 200);
});

// Response:
// {
//   "name": "John",
//   "age": 30
// }
```

## File Results

```csharp
// File download
app.MapGet("/files/{id}/download", async (string id) =>
{
    var bytes = await GetFileBytes(id);
    return Results.File(bytes, "application/pdf", "document.pdf");
});

// Stream file
app.MapGet("/large-file", () =>
{
    var stream = File.OpenRead("large-file.zip");
    return Results.Stream(stream, "application/zip", "download.zip");
});

// Physical file
app.MapGet("/static-file", () =>
{
    return Results.File("/path/to/file.pdf", "application/pdf");
});
```

## Redirect Results

```csharp
// Temporary redirect (302)
app.MapGet("/old-endpoint", () =>
{
    return Results.Redirect("/new-endpoint");
});

// Permanent redirect (301)
app.MapGet("/legacy", () =>
{
    return Results.RedirectPermanent("/current");
});

// Redirect to route
app.MapPost("/users", (CreateUserRequest request) =>
{
    var user = CreateUser(request);
    return Results.RedirectToRoute("GetUser", new { id = user.Id });
});

app.MapGet("/users/{id}", (string id) => GetUser(id))
    .WithName("GetUser");
```

## Content Results

```csharp
// Plain text
app.MapGet("/health", () =>
{
    return Results.Text("Healthy", "text/plain");
});

// HTML
app.MapGet("/page", () =>
{
    var html = "<html><body><h1>Hello</h1></body></html>";
    return Results.Content(html, "text/html");
});

// XML
app.MapGet("/xml-data", () =>
{
    var xml = "<root><item>value</item></root>";
    return Results.Content(xml, "application/xml");
});
```

## Validation Problem

```csharp
app.MapPost("/users", async (CreateUserRequest request, IValidator<CreateUserRequest> validator) =>
{
    var validationResult = await validator.ValidateAsync(request);

    if (!validationResult.IsValid)
    {
        var errors = validationResult.Errors
            .GroupBy(e => e.PropertyName)
            .ToDictionary(
                g => g.Key,
                g => g.Select(e => e.ErrorMessage).ToArray());

        return Results.ValidationProblem(errors);
    }

    var user = CreateUser(request);
    return Results.Created($"/users/{user.Id}", user);
});

// Response (400 Bad Request):
// {
//   "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
//   "title": "One or more validation errors occurred.",
//   "status": 400,
//   "errors": {
//     "Email": ["Email is required", "Email must be valid"],
//     "Name": ["Name is required"]
//   }
// }
```

## Problem Details (RFC 7807)

```csharp
// Generic problem
app.MapGet("/error", () =>
{
    return Results.Problem(
        detail: "Something went wrong",
        statusCode: 500);
});

// Custom problem details
app.MapGet("/not-found", () =>
{
    return Results.Problem(
        detail: "User not found",
        statusCode: 404,
        title: "User Not Found",
        type: "https://api.example.com/errors/user-not-found",
        instance: "/users/user-123");
});

// Response:
// {
//   "type": "https://api.example.com/errors/user-not-found",
//   "title": "User Not Found",
//   "status": 404,
//   "detail": "User not found",
//   "instance": "/users/user-123"
// }
```

## Challenge and Forbid

```csharp
// 401 Unauthorized (triggers authentication)
app.MapGet("/login-required", () =>
{
    return Results.Challenge();
});

// 403 Forbidden (authenticated but not authorized)
app.MapGet("/admin", () =>
{
    return Results.Forbid();
});

// Custom authentication scheme
app.MapGet("/oauth-login", () =>
{
    return Results.Challenge(
        authenticationSchemes: new[] { "OAuth" });
});
```

## Union Types (TypedResults)

```csharp
// Multiple possible return types
app.MapGet("/users/{id}", async (string id, IUserService userService)
    : Task<Results<Ok<User>, NotFound, BadRequest<string>>> =>
{
    if (string.IsNullOrEmpty(id))
    {
        return TypedResults.BadRequest("ID is required");
    }

    var user = await userService.GetUserByIdAsync(id);

    if (user is null)
    {
        return TypedResults.NotFound();
    }

    return TypedResults.Ok(user);
});

// OpenAPI knows all possible responses:
// - 200 OK with User
// - 404 Not Found
// - 400 Bad Request with string
```

## Custom Result

```csharp
public class CsvResult : IResult
{
    private readonly IEnumerable<object> _data;

    public CsvResult(IEnumerable<object> data)
    {
        _data = data;
    }

    public async Task ExecuteAsync(HttpContext context)
    {
        context.Response.ContentType = "text/csv";

        var csv = new StringBuilder();

        // Header
        var properties = _data.First().GetType().GetProperties();
        csv.AppendLine(string.Join(",", properties.Select(p => p.Name)));

        // Rows
        foreach (var item in _data)
        {
            var values = properties.Select(p => p.GetValue(item)?.ToString() ?? "");
            csv.AppendLine(string.Join(",", values));
        }

        await context.Response.WriteAsync(csv.ToString());
    }
}

// Extension method
public static class ResultsExtensions
{
    public static IResult Csv(this IResultExtensions _, IEnumerable<object> data)
    {
        return new CsvResult(data);
    }
}

// Usage
app.MapGet("/export", () =>
{
    var users = GetUsers();
    return Results.Extensions.Csv(users);
});
```

## Asynchronous Results

```csharp
// Task<IResult>
app.MapGet("/async-users", async (IUserService userService) : Task<IResult> =>
{
    var users = await userService.GetUsersAsync();
    return Results.Ok(users);
});

// Task<Results<...>>
app.MapGet("/typed-async", async (string id, IUserService userService)
    : Task<Results<Ok<User>, NotFound>> =>
{
    var user = await userService.GetUserByIdAsync(id);
    return user is not null ? TypedResults.Ok(user) : TypedResults.NotFound();
});
```

## Testing Result Types

```csharp
public class ResultTypesTests
{
    [Fact]
    public async Task GetUser_ReturnsOk()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        // Act
        var response = await client.GetAsync("/users/user-123");

        // Assert
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);

        var user = await response.Content.ReadFromJsonAsync<User>();
        Assert.NotNull(user);
    }

    [Fact]
    public async Task GetUser_NotFound_Returns404()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        // Act
        var response = await client.GetAsync("/users/nonexistent");

        // Assert
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
}
```

## Guidelines

**Minimal APIs:**
- Use `Results` for simple return types
- Use `TypedResults` for OpenAPI documentation
- Return `Task<Results<...>>` for multiple return types

**Controllers:**
- Use `ActionResult<T>` for typed responses
- Use `IActionResult` for dynamic responses
- Use `ProducesResponseType` for documentation

**Status Codes:**
- 200 OK: Success with body
- 201 Created: Resource created
- 204 No Content: Success without body
- 400 Bad Request: Validation error
- 401 Unauthorized: Authentication required
- 403 Forbidden: Insufficient permissions
- 404 Not Found: Resource not found
- 500 Internal Server Error: Server error

**Best Practices:**
- Be consistent (Results vs ActionResult)
- Document all return types
- Use Problem Details for errors
- Test all response types

## Benefits

Type-safe. Compile-time checks.

Documented. OpenAPI knows return types.

Consistent. Standard HTTP responses.

Testable. Easy to verify responses.

## Related

- [minimal-api-basics.md](./minimal-api-basics.md) - Minimal API endpoints
- [problem-details.md](../03-dotnet/problem-details.md) - RFC 7807 errors
- [api-error-responses.md](./api-error-responses.md) - Error responses
- [swagger-openapi.md](./swagger-openapi.md) - API documentation
