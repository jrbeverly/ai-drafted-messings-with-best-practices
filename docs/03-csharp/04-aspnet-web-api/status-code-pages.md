# Status Code Pages

Custom error pages. Status code handling. Problem Details. User-friendly error responses.

## Principle

Return appropriate HTTP status codes. Provide helpful error messages. Consistent error format.

## Status Code Reference

```csharp
// 2xx Success
200 OK                  // Request succeeded
201 Created             // Resource created
202 Accepted            // Request accepted, processing async
204 No Content          // Succeeded, no response body

// 3xx Redirection
301 Moved Permanently   // Resource moved
302 Found               // Temporary redirect
304 Not Modified        // Cached version valid

// 4xx Client Errors
400 Bad Request         // Invalid request
401 Unauthorized        // Authentication required
403 Forbidden           // Insufficient permissions
404 Not Found           // Resource not found
405 Method Not Allowed  // HTTP method not supported
409 Conflict            // Resource conflict
422 Unprocessable Entity // Validation failed
429 Too Many Requests   // Rate limit exceeded

// 5xx Server Errors
500 Internal Server Error // Server error
502 Bad Gateway         // Upstream error
503 Service Unavailable // Service temporarily down
504 Gateway Timeout     // Upstream timeout
```

## Basic Status Code Responses

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    // 200 OK
    [HttpGet("{id}")]
    public async Task<IActionResult> GetUser(string id)
    {
        var user = await _userService.GetUserByIdAsync(id);
        return user is not null ? Ok(user) : NotFound();
    }

    // 201 Created
    [HttpPost]
    public async Task<IActionResult> CreateUser(CreateUserRequest request)
    {
        var user = await _userService.CreateUserAsync(request);
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }

    // 204 No Content
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateUser(string id, UpdateUserRequest request)
    {
        await _userService.UpdateUserAsync(id, request);
        return NoContent();
    }

    // 202 Accepted (async processing)
    [HttpPost("batch")]
    public async Task<IActionResult> CreateUsersBatch(CreateUserRequest[] requests)
    {
        var jobId = await _userService.CreateUsersBatchAsync(requests);
        return Accepted($"/api/jobs/{jobId}", new { JobId = jobId });
    }
}
```

## Client Error Responses (4xx)

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    // 400 Bad Request
    [HttpPost]
    public async Task<IActionResult> CreateUser(CreateUserRequest request)
    {
        if (string.IsNullOrEmpty(request.Email))
        {
            return BadRequest(new { Error = "Email is required" });
        }

        var user = await _userService.CreateUserAsync(request);
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }

    // 401 Unauthorized
    [HttpGet("me")]
    public IActionResult GetCurrentUser()
    {
        if (!User.Identity?.IsAuthenticated ?? true)
        {
            return Unauthorized(new { Error = "Authentication required" });
        }

        var user = GetUserFromClaims();
        return Ok(user);
    }

    // 403 Forbidden
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteUser(string id)
    {
        if (!User.IsInRole("Admin"))
        {
            return Forbid();
        }

        await _userService.DeleteUserAsync(id);
        return NoContent();
    }

    // 404 Not Found
    [HttpGet("{id}")]
    public async Task<IActionResult> GetUser(string id)
    {
        var user = await _userService.GetUserByIdAsync(id);

        if (user is null)
        {
            return NotFound(new { Error = "User not found", Id = id });
        }

        return Ok(user);
    }

    // 409 Conflict
    [HttpPost]
    public async Task<IActionResult> CreateUser(CreateUserRequest request)
    {
        var existingUser = await _userService.GetUserByEmailAsync(request.Email);

        if (existingUser is not null)
        {
            return Conflict(new
            {
                Error = "User with this email already exists",
                Email = request.Email
            });
        }

        var user = await _userService.CreateUserAsync(request);
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }

    // 422 Unprocessable Entity
    [HttpPost]
    public async Task<IActionResult> CreateUser(CreateUserRequest request)
    {
        var validationResult = await _validator.ValidateAsync(request);

        if (!validationResult.IsValid)
        {
            return UnprocessableEntity(new
            {
                Error = "Validation failed",
                Errors = validationResult.Errors.Select(e => new
                {
                    e.PropertyName,
                    e.ErrorMessage
                })
            });
        }

        var user = await _userService.CreateUserAsync(request);
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }

    // 429 Too Many Requests
    [HttpGet]
    [EnableRateLimiting("fixed")]
    public async Task<IActionResult> GetUsers()
    {
        // Rate limiting handled by middleware
        // Returns 429 automatically when limit exceeded
        var users = await _userService.GetUsersAsync();
        return Ok(users);
    }
}
```

## Server Error Responses (5xx)

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    // 500 Internal Server Error
    [HttpGet("{id}")]
    public async Task<IActionResult> GetUser(string id)
    {
        try
        {
            var user = await _userService.GetUserByIdAsync(id);
            return user is not null ? Ok(user) : NotFound();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error retrieving user {UserId}", id);
            return Problem(
                detail: "An unexpected error occurred",
                statusCode: 500);
        }
    }

    // 503 Service Unavailable
    [HttpGet]
    public async Task<IActionResult> GetUsers()
    {
        if (!await _healthCheck.IsHealthyAsync())
        {
            return StatusCode(503, new
            {
                Error = "Service temporarily unavailable",
                RetryAfter = 60 // seconds
            });
        }

        var users = await _userService.GetUsersAsync();
        return Ok(users);
    }
}
```

## Problem Details (RFC 7807)

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<IActionResult> GetUser(string id)
    {
        var user = await _userService.GetUserByIdAsync(id);

        if (user is null)
        {
            return Problem(
                detail: $"User with ID '{id}' was not found",
                statusCode: 404,
                title: "User Not Found",
                type: "https://api.example.com/errors/user-not-found",
                instance: $"/api/users/{id}");
        }

        return Ok(user);
    }
}

// Response:
// {
//   "type": "https://api.example.com/errors/user-not-found",
//   "title": "User Not Found",
//   "status": 404,
//   "detail": "User with ID 'user-123' was not found",
//   "instance": "/api/users/user-123"
// }
```

## Custom Problem Details

```csharp
// Custom problem details with extensions
public class ValidationProblemDetails : ProblemDetails
{
    public Dictionary<string, string[]> Errors { get; set; } = new();
}

[HttpPost]
public async Task<IActionResult> CreateUser(CreateUserRequest request)
{
    var validationResult = await _validator.ValidateAsync(request);

    if (!validationResult.IsValid)
    {
        var errors = validationResult.Errors
            .GroupBy(e => e.PropertyName)
            .ToDictionary(
                g => g.Key,
                g => g.Select(e => e.ErrorMessage).ToArray());

        var problemDetails = new ValidationProblemDetails
        {
            Type = "https://api.example.com/errors/validation",
            Title = "Validation Failed",
            Status = 400,
            Detail = "One or more validation errors occurred",
            Instance = HttpContext.Request.Path,
            Errors = errors
        };

        return BadRequest(problemDetails);
    }

    var user = await _userService.CreateUserAsync(request);
    return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
}

// Response:
// {
//   "type": "https://api.example.com/errors/validation",
//   "title": "Validation Failed",
//   "status": 400,
//   "detail": "One or more validation errors occurred",
//   "instance": "/api/users",
//   "errors": {
//     "Email": ["Email is required", "Email must be valid"],
//     "Name": ["Name is required"]
//   }
// }
```

## Status Code Pages Middleware

```csharp
// Configure status code pages
var app = builder.Build();

app.UseStatusCodePages(async context =>
{
    var response = context.HttpContext.Response;

    if (response.StatusCode == 404)
    {
        await response.WriteAsJsonAsync(new
        {
            Error = "Resource not found",
            StatusCode = 404,
            Path = context.HttpContext.Request.Path.ToString()
        });
    }
    else if (response.StatusCode >= 400 && response.StatusCode < 500)
    {
        await response.WriteAsJsonAsync(new
        {
            Error = "Client error",
            StatusCode = response.StatusCode
        });
    }
    else if (response.StatusCode >= 500)
    {
        await response.WriteAsJsonAsync(new
        {
            Error = "Server error",
            StatusCode = response.StatusCode
        });
    }
});
```

## Minimal API Status Codes

```csharp
// Minimal API status code responses
app.MapGet("/users/{id}", async (string id, IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});

app.MapPost("/users", async (CreateUserRequest request, IUserService userService) =>
{
    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});

app.MapPut("/users/{id}", async (string id, UpdateUserRequest request, IUserService userService) =>
{
    await userService.UpdateUserAsync(id, request);
    return Results.NoContent();
});

// Custom status codes
app.MapGet("/health", () => Results.StatusCode(503));

// Problem Details
app.MapGet("/users/{id}", async (string id, IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);

    if (user is null)
    {
        return Results.Problem(
            detail: $"User with ID '{id}' not found",
            statusCode: 404,
            title: "User Not Found");
    }

    return Results.Ok(user);
});

// Validation Problem
app.MapPost("/users", async (
    CreateUserRequest request,
    IValidator<CreateUserRequest> validator) =>
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

    return Results.Ok();
});
```

## Content Negotiation for Errors

```csharp
// Return errors in requested format (JSON, XML, etc.)
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    [Produces("application/json", "application/xml")]
    public async Task<IActionResult> GetUser(string id)
    {
        var user = await _userService.GetUserByIdAsync(id);

        if (user is null)
        {
            // Respects Accept header for error format
            return NotFound(new
            {
                Error = "User not found",
                Id = id
            });
        }

        return Ok(user);
    }
}
```

## Testing Status Codes

```csharp
public class StatusCodeTests
{
    [Theory]
    [InlineData("/api/users/user-123", HttpStatusCode.OK)]
    [InlineData("/api/users/nonexistent", HttpStatusCode.NotFound)]
    public async Task GetUser_ReturnsCorrectStatusCode(string path, HttpStatusCode expectedStatus)
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        // Act
        var response = await client.GetAsync(path);

        // Assert
        Assert.Equal(expectedStatus, response.StatusCode);
    }

    [Fact]
    public async Task CreateUser_InvalidData_Returns400()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        var invalidRequest = new { Email = "", Name = "" };

        // Act
        var response = await client.PostAsJsonAsync("/api/users", invalidRequest);

        // Assert
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }

    [Fact]
    public async Task CreateUser_DuplicateEmail_Returns409()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        var request = new { Email = "existing@example.com", Name = "John" };

        // Act
        var response = await client.PostAsJsonAsync("/api/users", request);

        // Assert
        Assert.Equal(HttpStatusCode.Conflict, response.StatusCode);
    }
}
```

## Guidelines

**Status Code Selection:**
- 200 OK: Successful GET, PUT, PATCH
- 201 Created: Successful POST (resource created)
- 204 No Content: Successful DELETE, PUT (no body)
- 400 Bad Request: Invalid input, validation failure
- 401 Unauthorized: Authentication required
- 403 Forbidden: Insufficient permissions
- 404 Not Found: Resource doesn't exist
- 409 Conflict: Resource conflict (duplicate)
- 422 Unprocessable Entity: Semantic validation failure
- 500 Internal Server Error: Server error

**Error Messages:**
- User-friendly messages
- Include error code for machine parsing
- No sensitive information (stack traces, internal paths)
- Actionable guidance when possible
- Consistent error format

**Problem Details:**
- Use RFC 7807 format
- Include type, title, status, detail
- Optional instance (request path)
- Extensions for additional data

**Best Practices:**
- Return appropriate status codes
- Include Location header for 201 Created
- Use Problem Details for errors
- Log errors server-side
- Test all status codes

## Benefits

Clear. Explicit status communication.

Standard. HTTP status code conventions.

Debuggable. Helpful error messages.

Client-friendly. Machine-readable errors.

## Related

- [api-error-responses.md](./api-error-responses.md) - Error response format
- [error-codes-pattern.md](./error-codes-pattern.md) - Error code structure
- [problem-details.md](../03-dotnet/problem-details.md) - RFC 7807
- [global-error-handling.md](../05-aspnet-advanced/global-error-handling.md) - Exception handling
