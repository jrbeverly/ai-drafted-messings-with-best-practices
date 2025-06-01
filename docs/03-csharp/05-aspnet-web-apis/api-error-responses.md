# API Error Responses

All API errors use RFC 7807 Problem Details format, with automatic conversion from domain exceptions via middleware.

## Why It Matters

- Clients get a **standard, parseable** error structure for every failure
- Trace IDs link errors to logs and distributed traces
- Middleware eliminates error-formatting boilerplate in handlers

## Configuration

```csharp
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier;
        ctx.ProblemDetails.Instance = ctx.HttpContext.Request.Path;
    };
});
```

## Response Shape

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.4",
  "title": "Not Found",
  "status": 404,
  "detail": "Book with ID 'book_123' was not found",
  "instance": "/api/v1/books/book_123",
  "traceId": "0HN1GKFVQ9K7M:00000001"
}
```

Validation errors return `ValidationProblemDetails` with per-field `errors` dictionary.

## Status Code Quick Reference

| Code | Meaning | When |
|------|---------|------|
| 400 | Bad Request | Invalid input, business rule violation |
| 401 | Unauthorized | Missing/invalid authentication |
| 403 | Forbidden | Authenticated but insufficient permissions |
| 404 | Not Found | Resource does not exist |
| 409 | Conflict | Duplicate, version mismatch |
| 422 | Unprocessable Entity | Valid syntax, semantic errors |
| 500 | Internal Server Error | Unexpected server error |

## Manual Problem Details

```csharp
return Results.Problem(
    detail: "Resource is locked",
    statusCode: 423,
    title: "Locked",
    instance: context.Request.Path);
```

## Pitfalls to Avoid

- Returning plain strings or custom JSON shapes instead of Problem Details
- Omitting trace IDs (makes production debugging painful)
- Exposing stack traces or sensitive data in `detail`
- Inconsistent status codes across endpoints

## Related

- [domain-exceptions.md](./domain-exceptions.md) -- exception types and middleware
- [endpoint-filters.md](./endpoint-filters.md) -- validation error responses
