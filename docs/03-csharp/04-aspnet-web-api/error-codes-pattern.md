# Error Codes Pattern

Structured error codes. Machine-readable errors. Consistent error responses. Client error handling.

## Principle

Use structured error codes. Machine-readable format. Consistent across all endpoints. Enable client error handling.

## Error Code Structure

```csharp
// Error code format: DOMAIN_ENTITY_ERROR
// Examples:
// - USER_NOT_FOUND
// - PAYMENT_INSUFFICIENT_FUNDS
// - AUTH_INVALID_TOKEN
// - VALIDATION_REQUIRED_FIELD

public static class ErrorCodes
{
    // Authentication (AUTH_*)
    public const string AUTH_INVALID_TOKEN = "AUTH_INVALID_TOKEN";
    public const string AUTH_TOKEN_EXPIRED = "AUTH_TOKEN_EXPIRED";
    public const string AUTH_UNAUTHORIZED = "AUTH_UNAUTHORIZED";
    public const string AUTH_FORBIDDEN = "AUTH_FORBIDDEN";

    // User (USER_*)
    public const string USER_NOT_FOUND = "USER_NOT_FOUND";
    public const string USER_ALREADY_EXISTS = "USER_ALREADY_EXISTS";
    public const string USER_INACTIVE = "USER_INACTIVE";
    public const string USER_BANNED = "USER_BANNED";

    // Validation (VALIDATION_*)
    public const string VALIDATION_REQUIRED_FIELD = "VALIDATION_REQUIRED_FIELD";
    public const string VALIDATION_INVALID_FORMAT = "VALIDATION_INVALID_FORMAT";
    public const string VALIDATION_OUT_OF_RANGE = "VALIDATION_OUT_OF_RANGE";
    public const string VALIDATION_DUPLICATE = "VALIDATION_DUPLICATE";

    // Payment (PAYMENT_*)
    public const string PAYMENT_INSUFFICIENT_FUNDS = "PAYMENT_INSUFFICIENT_FUNDS";
    public const string PAYMENT_DECLINED = "PAYMENT_DECLINED";
    public const string PAYMENT_EXPIRED_CARD = "PAYMENT_EXPIRED_CARD";

    // Resource (RESOURCE_*)
    public const string RESOURCE_NOT_FOUND = "RESOURCE_NOT_FOUND";
    public const string RESOURCE_CONFLICT = "RESOURCE_CONFLICT";
    public const string RESOURCE_GONE = "RESOURCE_GONE";

    // System (SYSTEM_*)
    public const string SYSTEM_INTERNAL_ERROR = "SYSTEM_INTERNAL_ERROR";
    public const string SYSTEM_SERVICE_UNAVAILABLE = "SYSTEM_SERVICE_UNAVAILABLE";
    public const string SYSTEM_TIMEOUT = "SYSTEM_TIMEOUT";
}
```

## Error Response Format

```csharp
public record ApiError
{
    public required string Code { get; init; }
    public required string Message { get; init; }
    public string? Detail { get; init; }
    public Dictionary<string, string[]>? Errors { get; init; }
    public string? TraceId { get; init; }
    public DateTime Timestamp { get; init; } = DateTime.UtcNow;
}

// Example response
{
  "code": "USER_NOT_FOUND",
  "message": "User not found",
  "detail": "No user exists with ID 'user-123'",
  "traceId": "0HN1234567890",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

## Using Error Codes

```csharp
app.MapGet("/users/{id}", async (string id, IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);

    if (user is null)
    {
        return Results.Json(
            new ApiError
            {
                Code = ErrorCodes.USER_NOT_FOUND,
                Message = "User not found",
                Detail = $"No user exists with ID '{id}'"
            },
            statusCode: 404);
    }

    return Results.Ok(user);
});

app.MapPost("/users", async (
    CreateUserRequest request,
    IUserService userService,
    HttpContext context) =>
{
    try
    {
        var user = await userService.CreateUserAsync(request);
        return Results.Created($"/users/{user.Id}", user);
    }
    catch (DuplicateEmailException ex)
    {
        return Results.Json(
            new ApiError
            {
                Code = ErrorCodes.USER_ALREADY_EXISTS,
                Message = "User already exists",
                Detail = $"A user with email '{request.Email}' already exists",
                TraceId = context.TraceIdentifier
            },
            statusCode: 409);
    }
});
```

## Validation Error Codes

```csharp
app.MapPost("/users", async (
    CreateUserRequest request,
    IValidator<CreateUserRequest> validator,
    IUserService userService,
    HttpContext context) =>
{
    var validationResult = await validator.ValidateAsync(request);

    if (!validationResult.IsValid)
    {
        var errors = validationResult.Errors
            .GroupBy(e => e.PropertyName)
            .ToDictionary(
                g => g.Key,
                g => g.Select(e => e.ErrorMessage).ToArray());

        return Results.Json(
            new ApiError
            {
                Code = ErrorCodes.VALIDATION_REQUIRED_FIELD,
                Message = "Validation failed",
                Errors = errors,
                TraceId = context.TraceIdentifier
            },
            statusCode: 400);
    }

    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});

// Response:
{
  "code": "VALIDATION_REQUIRED_FIELD",
  "message": "Validation failed",
  "errors": {
    "Email": ["Email is required", "Email must be valid"],
    "Name": ["Name is required"]
  },
  "traceId": "0HN1234567890",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

## Custom Exceptions with Error Codes

```csharp
public abstract class ApiException : Exception
{
    public string ErrorCode { get; }
    public int StatusCode { get; }

    protected ApiException(string errorCode, string message, int statusCode)
        : base(message)
    {
        ErrorCode = errorCode;
        StatusCode = statusCode;
    }
}

public class UserNotFoundException : ApiException
{
    public UserNotFoundException(string userId)
        : base(
            ErrorCodes.USER_NOT_FOUND,
            "User not found",
            404)
    {
        Data["userId"] = userId;
    }
}

public class InsufficientFundsException : ApiException
{
    public InsufficientFundsException(decimal required, decimal available)
        : base(
            ErrorCodes.PAYMENT_INSUFFICIENT_FUNDS,
            "Insufficient funds",
            402)
    {
        Data["required"] = required;
        Data["available"] = available;
    }
}

// Usage
public async Task<User> GetUserAsync(string userId)
{
    var user = await _repository.GetByIdAsync(userId);

    if (user is null)
    {
        throw new UserNotFoundException(userId);
    }

    return user;
}
```

## Exception Handler Middleware

```csharp
public class ErrorCodeExceptionHandler : IExceptionHandler
{
    private readonly ILogger<ErrorCodeExceptionHandler> _logger;

    public ErrorCodeExceptionHandler(ILogger<ErrorCodeExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken cancellationToken)
    {
        if (exception is ApiException apiException)
        {
            _logger.LogWarning(
                exception,
                "API exception: {ErrorCode}",
                apiException.ErrorCode);

            context.Response.StatusCode = apiException.StatusCode;

            var error = new ApiError
            {
                Code = apiException.ErrorCode,
                Message = apiException.Message,
                Detail = apiException.Data.Count > 0
                    ? JsonSerializer.Serialize(apiException.Data)
                    : null,
                TraceId = context.TraceIdentifier
            };

            await context.Response.WriteAsJsonAsync(error, cancellationToken);

            return true;
        }

        // Unhandled exception
        _logger.LogError(exception, "Unhandled exception");

        context.Response.StatusCode = 500;

        var genericError = new ApiError
        {
            Code = ErrorCodes.SYSTEM_INTERNAL_ERROR,
            Message = "An internal server error occurred",
            TraceId = context.TraceIdentifier
        };

        await context.Response.WriteAsJsonAsync(genericError, cancellationToken);

        return true;
    }
}

// Register
builder.Services.AddExceptionHandler<ErrorCodeExceptionHandler>();

app.UseExceptionHandler();
```

## Error Code Documentation

```csharp
// Generate error code documentation
public class ErrorCodeDocumentation
{
    public static Dictionary<string, ErrorCodeInfo> GetErrorCodes()
    {
        return new Dictionary<string, ErrorCodeInfo>
        {
            [ErrorCodes.USER_NOT_FOUND] = new ErrorCodeInfo
            {
                Code = ErrorCodes.USER_NOT_FOUND,
                HttpStatus = 404,
                Message = "User not found",
                Description = "The requested user does not exist in the system",
                Example = new ApiError
                {
                    Code = ErrorCodes.USER_NOT_FOUND,
                    Message = "User not found",
                    Detail = "No user exists with ID 'user-123'"
                }
            },
            [ErrorCodes.VALIDATION_REQUIRED_FIELD] = new ErrorCodeInfo
            {
                Code = ErrorCodes.VALIDATION_REQUIRED_FIELD,
                HttpStatus = 400,
                Message = "Validation failed",
                Description = "One or more required fields are missing or invalid",
                Example = new ApiError
                {
                    Code = ErrorCodes.VALIDATION_REQUIRED_FIELD,
                    Message = "Validation failed",
                    Errors = new Dictionary<string, string[]>
                    {
                        ["Email"] = new[] { "Email is required" }
                    }
                }
            }
        };
    }
}

public record ErrorCodeInfo
{
    public required string Code { get; init; }
    public required int HttpStatus { get; init; }
    public required string Message { get; init; }
    public required string Description { get; init; }
    public required ApiError Example { get; init; }
}

// Expose error codes endpoint
app.MapGet("/api/error-codes", () =>
{
    var errorCodes = ErrorCodeDocumentation.GetErrorCodes();
    return Results.Ok(errorCodes);
});
```

## Client-Side Error Handling

```typescript
// TypeScript client
interface ApiError {
  code: string;
  message: string;
  detail?: string;
  errors?: Record<string, string[]>;
  traceId?: string;
  timestamp: string;
}

async function createUser(user: CreateUserRequest): Promise<User> {
  const response = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(user)
  });

  if (!response.ok) {
    const error: ApiError = await response.json();

    switch (error.code) {
      case 'USER_ALREADY_EXISTS':
        throw new DuplicateUserError(error.message);
      case 'VALIDATION_REQUIRED_FIELD':
        throw new ValidationError(error.errors);
      case 'AUTH_UNAUTHORIZED':
        // Redirect to login
        window.location.href = '/login';
        throw new UnauthorizedError();
      default:
        throw new ApiError(`API error: ${error.message}`);
    }
  }

  return response.json();
}
```

## Localized Error Messages

```csharp
public class LocalizedErrorCodes
{
    private readonly IStringLocalizer<ErrorMessages> _localizer;

    public LocalizedErrorCodes(IStringLocalizer<ErrorMessages> localizer)
    {
        _localizer = localizer;
    }

    public string GetMessage(string errorCode)
    {
        return _localizer[errorCode].Value;
    }
}

// Resources/ErrorMessages.en.resx
// USER_NOT_FOUND = "User not found"

// Resources/ErrorMessages.es.resx
// USER_NOT_FOUND = "Usuario no encontrado"

app.MapGet("/users/{id}", async (
    string id,
    IUserService userService,
    LocalizedErrorCodes errorCodes,
    HttpContext context) =>
{
    var user = await userService.GetUserByIdAsync(id);

    if (user is null)
    {
        return Results.Json(
            new ApiError
            {
                Code = ErrorCodes.USER_NOT_FOUND,
                Message = errorCodes.GetMessage(ErrorCodes.USER_NOT_FOUND),
                TraceId = context.TraceIdentifier
            },
            statusCode: 404);
    }

    return Results.Ok(user);
});
```

## Guidelines

**Error Code Format:**
- Prefix with domain (USER_, PAYMENT_, AUTH_)
- Use SCREAMING_SNAKE_CASE
- Be specific and descriptive
- Document all codes

**Error Response:**
- Include error code (machine-readable)
- Include message (human-readable)
- Include detail for context
- Include trace ID for debugging
- Include timestamp

**HTTP Status Codes:**
- 400: Validation errors
- 401: Authentication required
- 403: Insufficient permissions
- 404: Resource not found
- 409: Conflict (duplicate)
- 422: Unprocessable entity
- 500: Internal server error

**Client Handling:**
- Switch on error code (not message)
- Display localized message
- Handle specific codes differently
- Log errors with trace ID

## Benefits

Machine-readable. Clients can handle errors programmatically.

Consistent. Same format across all endpoints.

Debuggable. Trace IDs link to server logs.

Localized. Messages can be translated.

## Related

- [api-error-responses.md](./api-error-responses.md) - Error response format
- [problem-details.md](../03-dotnet/problem-details.md) - RFC 7807
- [exception-handling.md](./exception-handling.md) - Exception middleware
- [validation.md](./validation.md) - Input validation
