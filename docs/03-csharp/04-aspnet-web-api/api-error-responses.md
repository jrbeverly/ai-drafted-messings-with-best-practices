# API Error Responses

Consistent, machine-readable error format across all endpoints using Problem Details (RFC 7807) and structured error codes.

## Why It Matters

- Clients need predictable error structure to handle failures programmatically
- Trace IDs link client errors to server logs for debugging
- Consistent format reduces integration effort and support burden

## Key Patterns

**Standard error record:**

```csharp
public record ApiError
{
    public required string Code { get; init; }     // Machine-readable: "USER_NOT_FOUND"
    public required string Message { get; init; }  // Human-readable
    public string? Detail { get; init; }
    public Dictionary<string, string[]>? Errors { get; init; } // Validation errors
    public string? TraceId { get; init; }
}
```

**Global exception handler (IExceptionHandler):**

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();
app.UseExceptionHandler();

// In handler: map exception types to (statusCode, errorCode, message)
// NotFoundException => (404, "NOT_FOUND", ex.Message)
// ValidationException => (400, "VALIDATION_ERROR", ex.Message)
// _ => (500, "INTERNAL_ERROR", "An unexpected error occurred")
```

**Custom exception hierarchy:**

```csharp
public abstract class ApiException : Exception
{
    public string ErrorCode { get; }
    public int StatusCode { get; }
}
// Derive: NotFoundException, ConflictException, ValidationException
```

## Pitfalls to Avoid

- Exposing stack traces or internal paths in production responses
- Using generic "An error occurred" without error codes (not actionable)
- Forgetting `TraceId` in error responses (makes debugging impossible)
- Inconsistent error format between endpoints (some Problem Details, some ad-hoc)
- Logging at wrong level: 4xx = Warning, 5xx = Error
