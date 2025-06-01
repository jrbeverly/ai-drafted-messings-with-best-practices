# Error Code Catalog

Centralized error codes. Documentation. Categorization. Client reference.

## Principle

Maintain a single source of truth for error codes. Categorize by domain. Document meaning and resolution.

## Error Code Structure

```csharp
// Namespace-based organization
public static class ErrorCodes
{
    // Authentication errors (AUTH_*)
    public static class Auth
    {
        public const string INVALID_TOKEN = "AUTH_INVALID_TOKEN";
        public const string EXPIRED_TOKEN = "AUTH_EXPIRED_TOKEN";
        public const string MISSING_TOKEN = "AUTH_MISSING_TOKEN";
        public const string INVALID_CREDENTIALS = "AUTH_INVALID_CREDENTIALS";
        public const string ACCOUNT_LOCKED = "AUTH_ACCOUNT_LOCKED";
        public const string EMAIL_NOT_VERIFIED = "AUTH_EMAIL_NOT_VERIFIED";
    }

    // User errors (USER_*)
    public static class User
    {
        public const string NOT_FOUND = "USER_NOT_FOUND";
        public const string ALREADY_EXISTS = "USER_ALREADY_EXISTS";
        public const string INVALID_EMAIL = "USER_INVALID_EMAIL";
        public const string WEAK_PASSWORD = "USER_WEAK_PASSWORD";
        public const string DUPLICATE_EMAIL = "USER_DUPLICATE_EMAIL";
    }

    // Validation errors (VALIDATION_*)
    public static class Validation
    {
        public const string REQUIRED_FIELD = "VALIDATION_REQUIRED_FIELD";
        public const string INVALID_FORMAT = "VALIDATION_INVALID_FORMAT";
        public const string OUT_OF_RANGE = "VALIDATION_OUT_OF_RANGE";
        public const string INVALID_LENGTH = "VALIDATION_INVALID_LENGTH";
        public const string INVALID_TYPE = "VALIDATION_INVALID_TYPE";
    }

    // Payment errors (PAYMENT_*)
    public static class Payment
    {
        public const string INSUFFICIENT_FUNDS = "PAYMENT_INSUFFICIENT_FUNDS";
        public const string CARD_DECLINED = "PAYMENT_CARD_DECLINED";
        public const string EXPIRED_CARD = "PAYMENT_EXPIRED_CARD";
        public const string INVALID_AMOUNT = "PAYMENT_INVALID_AMOUNT";
        public const string PROCESSOR_ERROR = "PAYMENT_PROCESSOR_ERROR";
    }

    // Resource errors (RESOURCE_*)
    public static class Resource
    {
        public const string NOT_FOUND = "RESOURCE_NOT_FOUND";
        public const string ALREADY_EXISTS = "RESOURCE_ALREADY_EXISTS";
        public const string CONFLICT = "RESOURCE_CONFLICT";
        public const string LOCKED = "RESOURCE_LOCKED";
        public const string DELETED = "RESOURCE_DELETED";
    }

    // Rate limiting errors (RATE_*)
    public static class RateLimit
    {
        public const string EXCEEDED = "RATE_LIMIT_EXCEEDED";
        public const string TOO_MANY_REQUESTS = "RATE_LIMIT_TOO_MANY_REQUESTS";
        public const string QUOTA_EXCEEDED = "RATE_LIMIT_QUOTA_EXCEEDED";
    }

    // Internal errors (INTERNAL_*)
    public static class Internal
    {
        public const string ERROR = "INTERNAL_ERROR";
        public const string DATABASE_ERROR = "INTERNAL_DATABASE_ERROR";
        public const string SERVICE_UNAVAILABLE = "INTERNAL_SERVICE_UNAVAILABLE";
        public const string TIMEOUT = "INTERNAL_TIMEOUT";
    }
}
```

## Error Code Metadata

```csharp
// Error code with metadata
public record ErrorCodeInfo
{
    public required string Code { get; init; }
    public required string Message { get; init; }
    public required int StatusCode { get; init; }
    public string? Description { get; init; }
    public string? Resolution { get; init; }
    public string? DocumentationUrl { get; init; }
}

public static class ErrorCodeCatalog
{
    private static readonly Dictionary<string, ErrorCodeInfo> _catalog = new()
    {
        [ErrorCodes.Auth.INVALID_TOKEN] = new()
        {
            Code = ErrorCodes.Auth.INVALID_TOKEN,
            Message = "Invalid authentication token",
            StatusCode = 401,
            Description = "The provided authentication token is malformed or invalid",
            Resolution = "Obtain a new token by logging in again",
            DocumentationUrl = "https://docs.example.com/auth/tokens"
        },

        [ErrorCodes.Auth.EXPIRED_TOKEN] = new()
        {
            Code = ErrorCodes.Auth.EXPIRED_TOKEN,
            Message = "Authentication token has expired",
            StatusCode = 401,
            Description = "The authentication token is valid but has expired",
            Resolution = "Refresh your token or log in again",
            DocumentationUrl = "https://docs.example.com/auth/token-refresh"
        },

        [ErrorCodes.User.NOT_FOUND] = new()
        {
            Code = ErrorCodes.User.NOT_FOUND,
            Message = "User not found",
            StatusCode = 404,
            Description = "No user exists with the specified ID or email",
            Resolution = "Verify the user ID or email and try again",
            DocumentationUrl = "https://docs.example.com/users/errors"
        },

        [ErrorCodes.Payment.INSUFFICIENT_FUNDS] = new()
        {
            Code = ErrorCodes.Payment.INSUFFICIENT_FUNDS,
            Message = "Insufficient funds",
            StatusCode = 400,
            Description = "The account does not have enough funds for this transaction",
            Resolution = "Add funds to your account or use a different payment method",
            DocumentationUrl = "https://docs.example.com/payments/errors"
        },

        [ErrorCodes.RateLimit.EXCEEDED] = new()
        {
            Code = ErrorCodes.RateLimit.EXCEEDED,
            Message = "Rate limit exceeded",
            StatusCode = 429,
            Description = "Too many requests have been made in a short period",
            Resolution = "Wait before making additional requests. Check Retry-After header.",
            DocumentationUrl = "https://docs.example.com/rate-limits"
        }
    };

    public static ErrorCodeInfo GetInfo(string code)
    {
        return _catalog.TryGetValue(code, out var info)
            ? info
            : new ErrorCodeInfo
            {
                Code = code,
                Message = "An error occurred",
                StatusCode = 500
            };
    }

    public static IEnumerable<ErrorCodeInfo> GetAll() => _catalog.Values;

    public static IEnumerable<ErrorCodeInfo> GetByCategory(string category)
    {
        return _catalog.Values.Where(e => e.Code.StartsWith($"{category}_"));
    }
}
```

## Usage in API

```csharp
// Use error code catalog
app.MapGet("/users/{id}", async (string id, IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);

    if (user is null)
    {
        var errorInfo = ErrorCodeCatalog.GetInfo(ErrorCodes.User.NOT_FOUND);

        return Results.NotFound(new
        {
            errorInfo.Code,
            errorInfo.Message,
            Detail = $"User with ID '{id}' not found",
            errorInfo.Resolution,
            errorInfo.DocumentationUrl
        });
    }

    return Results.Ok(user);
});
```

## Error Documentation Endpoint

```csharp
// Public error code documentation
app.MapGet("/api/errors", () =>
{
    var errors = ErrorCodeCatalog.GetAll()
        .Select(e => new
        {
            e.Code,
            e.Message,
            e.StatusCode,
            e.Description,
            e.Resolution,
            e.DocumentationUrl
        });

    return Results.Ok(errors);
});

// Get errors by category
app.MapGet("/api/errors/{category}", (string category) =>
{
    var errors = ErrorCodeCatalog.GetByCategory(category.ToUpper())
        .Select(e => new
        {
            e.Code,
            e.Message,
            e.StatusCode,
            e.Description,
            e.Resolution
        });

    return errors.Any()
        ? Results.Ok(errors)
        : Results.NotFound();
});
```

## Localized Error Messages

```csharp
// Localized error catalog
public class LocalizedErrorCatalog
{
    private readonly IStringLocalizer<ErrorMessages> _localizer;

    public LocalizedErrorCatalog(IStringLocalizer<ErrorMessages> localizer)
    {
        _localizer = localizer;
    }

    public ErrorCodeInfo GetLocalizedInfo(string code, string? culture = null)
    {
        var baseInfo = ErrorCodeCatalog.GetInfo(code);

        return baseInfo with
        {
            Message = _localizer[code],
            Description = _localizer[$"{code}_DESCRIPTION"],
            Resolution = _localizer[$"{code}_RESOLUTION"]
        };
    }
}

// Resources/ErrorMessages.en.resx
// AUTH_INVALID_TOKEN = "Invalid authentication token"
// AUTH_INVALID_TOKEN_DESCRIPTION = "The provided token is malformed"
// AUTH_INVALID_TOKEN_RESOLUTION = "Obtain a new token by logging in"

// Resources/ErrorMessages.es.resx
// AUTH_INVALID_TOKEN = "Token de autenticación no válido"
// AUTH_INVALID_TOKEN_DESCRIPTION = "El token proporcionado está mal formado"
// AUTH_INVALID_TOKEN_RESOLUTION = "Obtenga un nuevo token iniciando sesión"
```

## Error Code Generator

```csharp
// Source generator for error codes (compile-time safety)
[ErrorCode("AUTH_INVALID_TOKEN", 401, "Invalid authentication token")]
[ErrorCode("USER_NOT_FOUND", 404, "User not found")]
[ErrorCode("PAYMENT_INSUFFICIENT_FUNDS", 400, "Insufficient funds")]
public partial class GeneratedErrorCodes
{
    // Generated at compile time:
    // public const string AUTH_INVALID_TOKEN = "AUTH_INVALID_TOKEN";
    // public const string USER_NOT_FOUND = "USER_NOT_FOUND";
    // etc.
}
```

## Validation with Error Codes

```csharp
// FluentValidation with error codes
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .WithErrorCode(ErrorCodes.Validation.REQUIRED_FIELD)
            .WithMessage("Email is required")
            .EmailAddress()
            .WithErrorCode(ErrorCodes.User.INVALID_EMAIL)
            .WithMessage("Email must be valid");

        RuleFor(x => x.Password)
            .NotEmpty()
            .WithErrorCode(ErrorCodes.Validation.REQUIRED_FIELD)
            .MinimumLength(8)
            .WithErrorCode(ErrorCodes.User.WEAK_PASSWORD)
            .WithMessage("Password must be at least 8 characters");
    }
}
```

## Error Code Analytics

```csharp
// Track error code frequency
public class ErrorCodeAnalytics
{
    private readonly ILogger<ErrorCodeAnalytics> _logger;

    public ErrorCodeAnalytics(ILogger<ErrorCodeAnalytics> logger)
    {
        _logger = logger;
    }

    public void LogError(string errorCode, string? detail = null)
    {
        _logger.LogWarning(
            "Error occurred: {ErrorCode}, Detail: {Detail}",
            errorCode,
            detail);

        // Track in metrics system
        // Metrics.Increment($"errors.{errorCode}");
    }
}

// Usage in exception handler
public class AnalyticsExceptionHandler : IExceptionHandler
{
    private readonly ErrorCodeAnalytics _analytics;

    public AnalyticsExceptionHandler(ErrorCodeAnalytics analytics)
    {
        _analytics = analytics;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var errorCode = exception is ApiException apiEx
            ? apiEx.ErrorCode
            : ErrorCodes.Internal.ERROR;

        _analytics.LogError(errorCode, exception.Message);

        var errorInfo = ErrorCodeCatalog.GetInfo(errorCode);

        context.Response.StatusCode = errorInfo.StatusCode;
        await context.Response.WriteAsJsonAsync(new
        {
            errorInfo.Code,
            errorInfo.Message,
            Detail = exception.Message,
            errorInfo.Resolution
        }, cancellationToken);

        return true;
    }
}
```

## Client-Side Error Handling

```typescript
// TypeScript client error handling
export enum ErrorCode {
  AUTH_INVALID_TOKEN = 'AUTH_INVALID_TOKEN',
  AUTH_EXPIRED_TOKEN = 'AUTH_EXPIRED_TOKEN',
  USER_NOT_FOUND = 'USER_NOT_FOUND',
  PAYMENT_INSUFFICIENT_FUNDS = 'PAYMENT_INSUFFICIENT_FUNDS',
  RATE_LIMIT_EXCEEDED = 'RATE_LIMIT_EXCEEDED',
}

export interface ApiError {
  code: ErrorCode;
  message: string;
  detail?: string;
  resolution?: string;
  documentationUrl?: string;
}

export async function handleApiError(error: ApiError) {
  switch (error.code) {
    case ErrorCode.AUTH_EXPIRED_TOKEN:
      // Redirect to login
      window.location.href = '/login';
      break;

    case ErrorCode.RATE_LIMIT_EXCEEDED:
      // Show retry message
      toast.error('Too many requests. Please wait and try again.');
      break;

    case ErrorCode.PAYMENT_INSUFFICIENT_FUNDS:
      // Show payment error with resolution
      toast.error(error.message);
      if (error.resolution) {
        toast.info(error.resolution);
      }
      break;

    default:
      // Generic error
      toast.error(error.message || 'An error occurred');
  }

  // Log to analytics
  analytics.trackError(error.code, error.message);
}
```

## Error Code Markdown Documentation

```markdown
# API Error Codes

## Authentication Errors (AUTH_*)

### AUTH_INVALID_TOKEN
- **Status:** 401 Unauthorized
- **Message:** Invalid authentication token
- **Description:** The provided authentication token is malformed or invalid
- **Resolution:** Obtain a new token by logging in again
- **Documentation:** [Authentication Guide](https://docs.example.com/auth/tokens)

### AUTH_EXPIRED_TOKEN
- **Status:** 401 Unauthorized
- **Message:** Authentication token has expired
- **Description:** The authentication token is valid but has expired
- **Resolution:** Refresh your token or log in again
- **Documentation:** [Token Refresh](https://docs.example.com/auth/token-refresh)

## User Errors (USER_*)

### USER_NOT_FOUND
- **Status:** 404 Not Found
- **Message:** User not found
- **Description:** No user exists with the specified ID or email
- **Resolution:** Verify the user ID or email and try again

### USER_DUPLICATE_EMAIL
- **Status:** 409 Conflict
- **Message:** Email already exists
- **Description:** A user with this email address already exists
- **Resolution:** Use a different email or log in to existing account
```

## Testing Error Codes

```csharp
public class ErrorCodeTests
{
    [Theory]
    [InlineData(ErrorCodes.Auth.INVALID_TOKEN, 401)]
    [InlineData(ErrorCodes.User.NOT_FOUND, 404)]
    [InlineData(ErrorCodes.Payment.INSUFFICIENT_FUNDS, 400)]
    public void ErrorCodeInfo_HasCorrectStatusCode(string errorCode, int expectedStatus)
    {
        // Arrange & Act
        var info = ErrorCodeCatalog.GetInfo(errorCode);

        // Assert
        Assert.Equal(expectedStatus, info.StatusCode);
        Assert.NotEmpty(info.Message);
        Assert.NotEmpty(info.Description);
    }

    [Fact]
    public void GetByCategory_Auth_ReturnsOnlyAuthErrors()
    {
        // Act
        var authErrors = ErrorCodeCatalog.GetByCategory("AUTH");

        // Assert
        Assert.All(authErrors, e => Assert.StartsWith("AUTH_", e.Code));
    }
}
```

## Guidelines

**Error Code Naming:**
- Use UPPERCASE_SNAKE_CASE
- Prefix with domain (AUTH_, USER_, PAYMENT_)
- Descriptive and specific (INVALID_TOKEN not BAD_AUTH)

**Catalog Maintenance:**
- Single source of truth
- Document all error codes
- Include resolution steps
- Link to detailed docs

**Versioning:**
- Don't remove error codes (deprecate instead)
- Add new codes as needed
- Version documentation with API

**Client Communication:**
- Expose error catalog endpoint
- Generate TypeScript types from C# codes
- Provide resolution guidance
- Link to documentation

## Benefits

Centralized. Single source of truth.

Documented. Clear descriptions and resolutions.

Type-safe. Compile-time checking.

Client-friendly. Machine-readable codes.

## Related

- [error-codes-pattern.md](./error-codes-pattern.md) - Error code structure
- [api-error-responses.md](./api-error-responses.md) - Error response format
- [problem-details.md](../03-dotnet/problem-details.md) - RFC 7807
- [localization.md](../05-aspnet-advanced/localization.md) - Localized messages
