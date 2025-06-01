# CORS (Cross-Origin Resource Sharing)

Control which origins can call your API from a browser.

## Why It Matters

- Browsers block cross-origin requests by default -- CORS headers opt in
- Misconfigured CORS is either a security hole or a wall of opaque errors
- SignalR and credentials require explicit origin lists (no wildcards)

## Key Recommendations

- **Middleware order matters**: `UseRouting()` then `UseCors()` then `UseAuthentication()` then endpoints
- **Never use `AllowAnyOrigin` in production** -- specify exact origins:
  ```csharp
  builder.Services.AddCors(options =>
  {
      options.AddDefaultPolicy(policy =>
          policy.WithOrigins("https://app.example.com")
                .AllowAnyMethod()
                .AllowAnyHeader());
  });
  ```
- **Credentials require specific origins** -- `AllowAnyOrigin()` + `AllowCredentials()` throws at runtime
- **Store origins in config** (`appsettings.json` or env vars), vary per environment
- **Use named policies** for different endpoint groups (`PublicApi`, `RestrictedApi`)
- **Cache preflight responses**: `.SetPreflightMaxAge(TimeSpan.FromHours(1))`
- **Dynamic subdomains**: use `SetIsOriginAllowed(origin => ...)` with careful validation
- **Expose custom response headers** with `.WithExposedHeaders("X-Total-Count")`

## Development Setup

```csharp
if (builder.Environment.IsDevelopment())
    policy.WithOrigins("http://localhost:3000", "http://localhost:5173");
```

## Pitfalls to Avoid

- Placing `UseCors()` after `MapControllers()` (CORS never runs)
- Origin mismatch: scheme, host, and port must match exactly
- Forgetting `AllowCredentials()` when the frontend sends `credentials: 'include'`
- Not testing the OPTIONS preflight separately from the actual request
