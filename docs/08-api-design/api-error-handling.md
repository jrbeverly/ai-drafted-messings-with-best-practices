# API Error Handling

Consistent error responses. Problem details (RFC 7807). Validation errors. Logging and monitoring.

## Principle

Consistent error format. Actionable messages. Appropriate status codes. Never expose internals.

## Problem Details (RFC 7807)

Standard error format:

```csharp
// ProblemDetails response
public record ProblemDetailsResponse
{
    public required string Type { get; init; }          // URI identifying the error type
    public required string Title { get; init; }         // Short, human-readable summary
    public required int Status { get; init; }           // HTTP status code
    public required string Detail { get; init; }        // Human-readable explanation
    public required string Instance { get; init; }      // URI identifying this occurrence
    public Dictionary<string, object>? Extensions { get; init; }  // Additional details
}

// Example usage
public static IResult NotFound(string resource, string id)
{
    return Results.Problem(
        type: "https://api.example.com/errors/not-found",
        title: "Resource not found",
        status: StatusCodes.Status404NotFound,
        detail: $"{resource} with ID '{id}' was not found",
        instance: $"/users/{id}");
}
```

Built-in Problem Details:

```csharp
// Use ASP.NET Core built-in support
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        // Add request ID to all errors
        context.ProblemDetails.Extensions["requestId"] = context.HttpContext.TraceIdentifier;

        // Add timestamp
        context.ProblemDetails.Extensions["timestamp"] = DateTime.UtcNow;
    };
});
```

## Common Error Responses

Not Found (404):

```csharp
public static async Task<IResult> GetUser(
    string id,
    IUserService userService)
{
    var user = await userService.GetUserByIdAsync(id);

    if (user == null)
    {
        return Results.Problem(
            type: "https://api.example.com/errors/user-not-found",
            title: "User not found",
            status: 404,
            detail: $"User with ID '{id}' does not exist",
            instance: $"/users/{id}");
    }

    return Results.Ok(user);
}
```

Bad Request (400):

```csharp
public static async Task<IResult> CreateUser(
    [FromBody] CreateUserRequest? request,
    IUserService userService)
{
    if (request == null)
    {
        return Results.Problem(
            type: "https://api.example.com/errors/invalid-request",
            title: "Invalid request",
            status: 400,
            detail: "Request body is required",
            instance: "/users");
    }

    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
}
```

Conflict (409):

```csharp
public static async Task<IResult> CreateUser(
    [FromBody] CreateUserRequest request,
    IUserService userService)
{
    var existingUser = await userService.GetUserByEmailAsync(request.Email);

    if (existingUser != null)
    {
        return Results.Problem(
            type: "https://api.example.com/errors/duplicate-email",
            title: "Duplicate email",
            status: 409,
            detail: $"A user with email '{request.Email}' already exists",
            instance: "/users");
    }

    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
}
```

## Validation Errors

FluentValidation integration:

```csharp
// Validator
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Email must be a valid email address");

        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .MinimumLength(2).WithMessage("Name must be at least 2 characters")
            .MaximumLength(100).WithMessage("Name must not exceed 100 characters");
    }
}

// Validation filter
public class ValidationFilter<T> : IEndpointFilter
{
    private readonly IValidator<T> _validator;

    public ValidationFilter(IValidator<T> validator)
    {
        _validator = validator;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var request = context.Arguments.OfType<T>().FirstOrDefault();

        if (request == null)
            return await next(context);

        var validationResult = await _validator.ValidateAsync(request);

        if (!validationResult.IsValid)
        {
            return Results.ValidationProblem(
                validationResult.ToDictionary(),
                type: "https://api.example.com/errors/validation",
                title: "Validation failed",
                detail: "One or more validation errors occurred");
        }

        return await next(context);
    }
}

// Extension method
public static RouteHandlerBuilder WithValidation<T>(
    this RouteHandlerBuilder builder) where T : class
{
    return builder.AddEndpointFilter<ValidationFilter<T>>();
}

// Usage
app.MapPost("/users", CreateUser.HandleAsync)
    .WithValidation<CreateUserRequest>();
```

Validation error response:

```json
{
  "type": "https://api.example.com/errors/validation",
  "title": "Validation failed",
  "status": 400,
  "detail": "One or more validation errors occurred",
  "instance": "/users",
  "errors": {
    "email": [
      "Email is required",
      "Email must be a valid email address"
    ],
    "name": [
      "Name must be at least 2 characters"
    ]
  },
  "requestId": "0HN1GKFVQKQV2:00000001",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

## Exception Handling Middleware

Global exception handler:

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "Unhandled exception occurred");

        var (status, type, title, detail) = exception switch
        {
            NotFoundException ex => (404,
                "https://api.example.com/errors/not-found",
                "Resource not found",
                ex.Message),

            ValidationException ex => (400,
                "https://api.example.com/errors/validation",
                "Validation failed",
                ex.Message),

            ConflictException ex => (409,
                "https://api.example.com/errors/conflict",
                "Conflict",
                ex.Message),

            UnauthorizedAccessException => (401,
                "https://api.example.com/errors/unauthorized",
                "Unauthorized",
                "Authentication is required"),

            ForbiddenException ex => (403,
                "https://api.example.com/errors/forbidden",
                "Forbidden",
                ex.Message),

            _ => (500,
                "https://api.example.com/errors/server-error",
                "Internal server error",
                "An unexpected error occurred")
        };

        context.Response.StatusCode = status;
        await context.Response.WriteAsJsonAsync(new ProblemDetailsResponse
        {
            Type = type,
            Title = title,
            Status = status,
            Detail = detail,
            Instance = context.Request.Path,
            Extensions = new Dictionary<string, object>
            {
                ["requestId"] = context.TraceIdentifier,
                ["timestamp"] = DateTime.UtcNow
            }
        }, cancellationToken);

        return true;
    }
}

// Register
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
app.UseExceptionHandler(options => { });
```

## Custom Exception Types

Domain exceptions:

```csharp
// Base exception
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
    protected DomainException(string message, Exception innerException)
        : base(message, innerException) { }
}

// Specific exceptions
public class NotFoundException : DomainException
{
    public NotFoundException(string resource, string id)
        : base($"{resource} with ID '{id}' was not found")
    {
        Resource = resource;
        Id = id;
    }

    public string Resource { get; }
    public string Id { get; }
}

public class DuplicateException : DomainException
{
    public DuplicateException(string resource, string field, string value)
        : base($"{resource} with {field} '{value}' already exists")
    {
        Resource = resource;
        Field = field;
        Value = value;
    }

    public string Resource { get; }
    public string Field { get; }
    public string Value { get; }
}

public class ValidationException : DomainException
{
    public ValidationException(string message, Dictionary<string, string[]> errors)
        : base(message)
    {
        Errors = errors;
    }

    public Dictionary<string, string[]> Errors { get; }
}

// Usage
public async Task<User> CreateUserAsync(CreateUserRequest request)
{
    var existing = await _repository.GetByEmailAsync(request.Email);

    if (existing != null)
    {
        throw new DuplicateException("User", "email", request.Email);
    }

    // Create user...
}
```

## Error Logging

Structured logging:

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken cancellationToken)
    {
        // Log with structured properties
        _logger.LogError(exception,
            "Error processing request {Method} {Path}. RequestId: {RequestId}, User: {UserId}",
            context.Request.Method,
            context.Request.Path,
            context.TraceIdentifier,
            context.User.FindFirst("sub")?.Value ?? "anonymous");

        // Don't log validation errors as errors (they're client mistakes)
        if (exception is ValidationException)
        {
            _logger.LogWarning(exception,
                "Validation failed for {Method} {Path}",
                context.Request.Method,
                context.Request.Path);
        }

        // Return error response...
    }
}
```

## Client Error vs Server Error

Distinguish client and server errors:

```csharp
public async ValueTask<bool> TryHandleAsync(
    HttpContext context,
    Exception exception,
    CancellationToken cancellationToken)
{
    var isClientError = exception is
        NotFoundException or
        ValidationException or
        DuplicateException or
        UnauthorizedAccessException or
        ForbiddenException;

    if (isClientError)
    {
        // Log as warning (client's fault)
        _logger.LogWarning(exception,
            "Client error: {ExceptionType}",
            exception.GetType().Name);
    }
    else
    {
        // Log as error (our fault, needs investigation)
        _logger.LogError(exception,
            "Server error: {ExceptionType}",
            exception.GetType().Name);
    }

    // Return appropriate response...
}
```

## Rate Limit Errors

Handle rate limiting:

```csharp
public class RateLimitMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        if (IsRateLimited(context))
        {
            context.Response.StatusCode = 429;
            context.Response.Headers["Retry-After"] = "60";

            await context.Response.WriteAsJsonAsync(new ProblemDetailsResponse
            {
                Type = "https://api.example.com/errors/rate-limit",
                Title = "Too many requests",
                Status = 429,
                Detail = "Rate limit exceeded. Please try again in 60 seconds.",
                Instance = context.Request.Path,
                Extensions = new Dictionary<string, object>
                {
                    ["retryAfter"] = 60,
                    ["limit"] = 100,
                    ["remaining"] = 0,
                    ["resetAt"] = DateTime.UtcNow.AddMinutes(1)
                }
            });

            return;
        }

        await _next(context);
    }
}
```

## Async Operation Errors

Handle long-running operations:

```csharp
// Start async operation
POST /operations
{
  "type": "import",
  "fileUrl": "https://..."
}

// Response
201 Created
Location: /operations/op_123
{
  "id": "op_123",
  "status": "processing",
  "createdAt": "2024-01-15T10:30:00Z"
}

// Check status
GET /operations/op_123

// Error response
{
  "id": "op_123",
  "status": "failed",
  "error": {
    "type": "https://api.example.com/errors/import-failed",
    "title": "Import failed",
    "detail": "Invalid file format. Expected CSV, got JSON.",
    "timestamp": "2024-01-15T10:35:00Z"
  },
  "createdAt": "2024-01-15T10:30:00Z",
  "completedAt": "2024-01-15T10:35:00Z"
}
```

## Guidelines

**Error Format:**
- Use RFC 7807 Problem Details
- Include type, title, status, detail, instance
- Add requestId and timestamp
- Never expose stack traces in production

**Status Codes:**
- 400 Bad Request - Invalid input
- 401 Unauthorized - Not authenticated
- 403 Forbidden - Not authorized
- 404 Not Found - Resource doesn't exist
- 409 Conflict - Resource conflict
- 422 Unprocessable Entity - Validation failed
- 429 Too Many Requests - Rate limited
- 500 Internal Server Error - Server error
- 503 Service Unavailable - Temporary unavailable

**Validation:**
- Return field-specific errors
- Use consistent error format
- Include field name and messages
- Validate early (fail fast)

**Logging:**
- Log all server errors (500s)
- Log client errors as warnings
- Include request context
- Use structured logging

**Security:**
- Don't expose internal details
- Sanitize error messages
- Don't leak sensitive data
- Consistent timing for security checks

## Benefits

Consistency. Same format across all errors.

Actionable. Clear messages help clients fix issues.

Debuggable. RequestId and timestamps for investigation.

Standard. RFC 7807 compliance.

## Related

- [rest-api-principles.md](./rest-api-principles.md) - API design patterns
- [api-versioning.md](./api-versioning.md) - Version management
- [logging-structured-serilog.md](../03-csharp/03-dotnet/logging-structured-serilog.md) - Structured logging
