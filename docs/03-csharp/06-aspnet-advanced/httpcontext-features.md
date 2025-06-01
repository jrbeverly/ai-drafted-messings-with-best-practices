# HttpContext Features

Access low-level HTTP primitives and extend HttpContext with custom feature interfaces.

## Why It Matters

- Feature collections provide access to connection, TLS, WebSocket, and request lifetime details
- `IHttpContextAccessor` lets services outside the pipeline access the current request
- Custom features enable middleware-to-endpoint data sharing without coupling

## Key Recommendations

- **Register `IHttpContextAccessor`** when services need HttpContext:
  ```csharp
  builder.Services.AddHttpContextAccessor();
  ```
- **Always null-check** `_httpContextAccessor.HttpContext` -- it is null outside request scope
- **Use `HttpContext.Items`** for per-request data sharing between middleware and endpoints (dictionary keyed by `object`)
- **Prefer strongly-typed services** over `HttpContext.Items` when possible

## Common Built-In Features

| Feature Interface | Provides |
|---|---|
| `IHttpRequestFeature` | Raw method, path, query, protocol, headers, body |
| `IHttpResponseFeature` | Status code, `OnStarting` / `OnCompleted` callbacks |
| `IHttpConnectionFeature` | Remote/local IP and port |
| `ITlsConnectionFeature` | Client certificate |
| `IHttpWebSocketFeature` | WebSocket detection and acceptance |
| `IHttpRequestLifetimeFeature` | `RequestAborted` cancellation token |

## Custom Features

```csharp
// Define
public interface IRequestTimingFeature { DateTime StartTime { get; } TimeSpan Duration { get; } }

// Set in middleware
context.Features.Set<IRequestTimingFeature>(new RequestTimingFeature());

// Read in endpoint
var timing = context.Features.Get<IRequestTimingFeature>();
```

## Pitfalls to Avoid

- Storing references to `HttpContext` beyond the request lifetime (it gets disposed)
- Overusing `IHttpContextAccessor` -- it has a performance cost via `AsyncLocal`
- Forgetting to check for null when accessing optional features
- Using `HttpContext.Items` with string keys without a naming convention (collision risk)
