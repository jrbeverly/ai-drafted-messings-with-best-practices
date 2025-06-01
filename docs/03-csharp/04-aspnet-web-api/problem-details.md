# Problem Details (RFC 7807)

Standardized error response format for HTTP APIs. Machine-readable error information. Consistent error structure.

## Principle

RFC 7807 standard for error responses. Include type, title, status, detail, and instance. Consistent across all endpoints.

## Basic Problem Details

```csharp
app.MapGet("/users/{id}", async (
    string id,
    IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);

    if (user is null)
    {
        return Results.Problem(
            type: "https://api.example.com/errors/not-found",
            title: "User Not Found",
            status: StatusCodes.Status404NotFound,
            detail: $"User with ID '{id}' does not exist",
            instance: $"/users/{id}");
    }

    return Results.Ok(user);
});

// Response:
// {
//   "type": "https://api.example.com/errors/not-found",
//   "title": "User Not Found",
//   "status": 404,
//   "detail": "User with ID '123' does not exist",
//   "instance": "/users/123"
// }
```

## Validation Problem Details

```csharp
app.MapPost("/users", async (
    CreateUserRequest request,
    IValidator<CreateUserRequest> validator,
    IUserService userService) =>
{
    var validationResult = await validator.ValidateAsync(request);

    if (!validationResult.IsValid)
    {
        return Results.ValidationProblem(
            validationResult.ToDictionary(),
            type: "https://api.example.com/errors/validation",
            title: "Validation Failed");
    }

    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});

// Response:
// {
//   "type": "https://api.example.com/errors/validation",
//   "title": "Validation Failed",
//   "status": 400,
//   "errors": {
//     "Email": ["Email is required", "Email must be valid"],
//     "Name": ["Name must be between 2 and 100 characters"]
//   }
// }
```

## Custom Problem Details

```csharp
public record ApiProblemDetails : ProblemDetails
{
    public string ErrorCode { get; init; } = string.Empty;
    public Dictionary<string, object>? Extensions { get; init; }
}

app.MapPost("/users", async (
    CreateUserRequest request,
    IUserService userService) =>
{
    try
    {
        var user = await userService.CreateUserAsync(request);
        return Results.Created($"/users/{user.Id}", user);
    }
    catch (DuplicateEmailException ex)
    {
        var problem = new ApiProblemDetails
        {
            Type = "https://api.example.com/errors/duplicate-email",
            Title = "Duplicate Email",
            Status = StatusCodes.Status409Conflict,
            Detail = ex.Message,
            ErrorCode = "USER_001",
            Extensions = new Dictionary<string, object>
            {
                ["email"] = request.Email,
                ["suggestedAction"] = "Use a different email address or log in"
            }
        };

        return Results.Json(problem, statusCode: 409);
    }
});
```

## Global Exception Handler

```csharp
// Configure problem details
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Instance =
            $"{context.HttpContext.Request.Method} {context.HttpContext.Request.Path}";

        context.ProblemDetails.Extensions["traceId"] =
            context.HttpContext.TraceIdentifier;

        context.ProblemDetails.Extensions["timestamp"] =
            DateTime.UtcNow.ToString("o");
    };
});

app.UseExceptionHandler();

// Custom exception handler
app.UseExceptionHandler(exceptionHandlerApp =>
{
    exceptionHandlerApp.Run(async context =>
    {
        var exceptionHandlerFeature =
            context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionHandlerFeature?.Error;

        var problemDetails = exception switch
        {
            NotFoundException ex => new ProblemDetails
            {
                Type = "https://api.example.com/errors/not-found",
                Title = "Resource Not Found",
                Status = StatusCodes.Status404NotFound,
                Detail = ex.Message
            },

            ValidationException ex => new ProblemDetails
            {
                Type = "https://api.example.com/errors/validation",
                Title = "Validation Failed",
                Status = StatusCodes.Status400BadRequest,
                Detail = ex.Message
            },

            UnauthorizedAccessException => new ProblemDetails
            {
                Type = "https://api.example.com/errors/unauthorized",
                Title = "Unauthorized",
                Status = StatusCodes.Status401Unauthorized,
                Detail = "Authentication is required"
            },

            _ => new ProblemDetails
            {
                Type = "https://api.example.com/errors/internal-server-error",
                Title = "Internal Server Error",
                Status = StatusCodes.Status500InternalServerError,
                Detail = "An unexpected error occurred"
            }
        };

        problemDetails.Instance = $"{context.Request.Method} {context.Request.Path}";
        problemDetails.Extensions["traceId"] = context.TraceIdentifier;

        context.Response.StatusCode = problemDetails.Status ?? 500;
        await context.Response.WriteAsJsonAsync(problemDetails);
    });
});
```

## Exception Types

```csharp
// Custom exceptions
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}

public class DuplicateEmailException : Exception
{
    public string Email { get; }

    public DuplicateEmailException(string email)
        : base($"Email '{email}' is already in use")
    {
        Email = email;
    }
}

public class InsufficientPermissionException : Exception
{
    public string RequiredPermission { get; }

    public InsufficientPermissionException(string permission)
        : base($"Insufficient permission: {permission} required")
    {
        RequiredPermission = permission;
    }
}

// Usage in service
public class UserService : IUserService
{
    public async Task<User> GetUserByIdAsync(string id)
    {
        var user = await _repository.GetByIdAsync(id);

        if (user is null)
        {
            throw new NotFoundException($"User with ID '{id}' not found");
        }

        return user;
    }

    public async Task<User> CreateUserAsync(CreateUserRequest request)
    {
        var existing = await _repository.GetByEmailAsync(request.Email);

        if (existing is not null)
        {
            throw new DuplicateEmailException(request.Email);
        }

        return await _repository.CreateAsync(request);
    }
}
```

## Result Pattern (Alternative)

```csharp
// Result type to avoid exceptions
public record Result<T>
{
    public T? Value { get; init; }
    public ProblemDetails? Error { get; init; }
    public bool IsSuccess => Error is null;

    public static Result<T> Success(T value) => new() { Value = value };
    public static Result<T> Failure(ProblemDetails error) => new() { Error = error };
}

// Service returns Result
public class UserService : IUserService
{
    public async Task<Result<User>> GetUserByIdAsync(string id)
    {
        var user = await _repository.GetByIdAsync(id);

        if (user is null)
        {
            return Result<User>.Failure(new ProblemDetails
            {
                Type = "https://api.example.com/errors/not-found",
                Title = "User Not Found",
                Status = 404,
                Detail = $"User with ID '{id}' not found"
            });
        }

        return Result<User>.Success(user);
    }
}

// Endpoint handles Result
app.MapGet("/users/{id}", async (
    string id,
    IUserService userService) =>
{
    var result = await userService.GetUserByIdAsync(id);

    return result.IsSuccess
        ? Results.Ok(result.Value)
        : Results.Problem(result.Error);
});
```

## Common Error Types

```csharp
public static class ProblemTypes
{
    private const string BaseUrl = "https://api.example.com/errors";

    public static class NotFound
    {
        public const string Type = $"{BaseUrl}/not-found";
        public const string Title = "Resource Not Found";
        public const int Status = 404;
    }

    public static class Validation
    {
        public const string Type = $"{BaseUrl}/validation";
        public const string Title = "Validation Failed";
        public const int Status = 400;
    }

    public static class Unauthorized
    {
        public const string Type = $"{BaseUrl}/unauthorized";
        public const string Title = "Unauthorized";
        public const int Status = 401;
    }

    public static class Forbidden
    {
        public const string Type = $"{BaseUrl}/forbidden";
        public const string Title = "Forbidden";
        public const int Status = 403;
    }

    public static class Conflict
    {
        public const string Type = $"{BaseUrl}/conflict";
        public const string Title = "Resource Conflict";
        public const int Status = 409;
    }

    public static class InternalServerError
    {
        public const string Type = $"{BaseUrl}/internal-server-error";
        public const string Title = "Internal Server Error";
        public const int Status = 500;
    }
}

// Usage
app.MapGet("/users/{id}", async (string id, IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);

    if (user is null)
    {
        return Results.Problem(
            type: ProblemTypes.NotFound.Type,
            title: ProblemTypes.NotFound.Title,
            status: ProblemTypes.NotFound.Status,
            detail: $"User '{id}' not found");
    }

    return Results.Ok(user);
});
```

## Development vs Production

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler(exceptionHandlerApp =>
    {
        exceptionHandlerApp.Run(async context =>
        {
            var exceptionHandlerFeature =
                context.Features.Get<IExceptionHandlerFeature>();
            var exception = exceptionHandlerFeature?.Error;

            var problemDetails = new ProblemDetails
            {
                Type = ProblemTypes.InternalServerError.Type,
                Title = ProblemTypes.InternalServerError.Title,
                Status = ProblemTypes.InternalServerError.Status,
                Detail = "An unexpected error occurred",  // Generic message
                Instance = $"{context.Request.Method} {context.Request.Path}"
            };

            problemDetails.Extensions["traceId"] = context.TraceIdentifier;

            // Log exception details (not exposed to client)
            var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
            logger.LogError(exception, "Unhandled exception occurred");

            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(problemDetails);
        });
    });
}
```

## Guidelines

**Problem Details Structure:**
- `type`: URL identifying the error type
- `title`: Short, human-readable summary
- `status`: HTTP status code
- `detail`: Specific explanation
- `instance`: Request that caused the error

**Best Practices:**
- Use RFC 7807 format consistently
- Include trace IDs for debugging
- Generic errors in production (no stack traces)
- Detailed errors in development
- Document error types in API docs

**Error Type URLs:**
- Use your domain (`https://api.example.com/errors/...`)
- Make URLs descriptive (`not-found`, `validation`)
- URLs don't need to resolve (just identifiers)
- Keep consistent across API versions

**Extensions:**
- Add custom fields sparingly
- Include actionable information
- Avoid sensitive data
- Consider client needs

## Benefits

Standard. RFC 7807 compliance.

Consistent. Same format for all errors.

Machine-readable. Easy for clients to parse.

Debuggable. Includes trace IDs.

## Related

- [api-error-handling.md](../../08-api-design/api-error-handling.md) - Error handling strategies
- [minimal-api-basics.md](./minimal-api-basics.md) - Endpoint definition
- [request-validation.md](./request-validation.md) - Validation errors
- [logging-structured-serilog.md](../03-dotnet/logging-structured-serilog.md) - Error logging
