# CORS Configuration

Cross-Origin Resource Sharing controls which domains can call your API from browsers.

## Why It Matters

- Without CORS, browsers block cross-origin API calls from your frontend
- Misconfigured CORS is a security vulnerability (wildcard origins in production)
- Different endpoints may need different policies (public API vs admin)

## Key Patterns

```csharp
// Named policies
builder.Services.AddCors(options =>
{
    options.AddPolicy("Production", policy =>
        policy.WithOrigins("https://app.example.com")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Content-Type", "Authorization")
              .AllowCredentials()
              .SetPreflightMaxAge(TimeSpan.FromHours(1)));
});

app.UseCors("Production");

// Per-endpoint/group
app.MapGroup("/api/public").RequireCors("PublicApi");
app.MapGet("/health", GetHealth).DisableCors();
```

**Config-driven origins:** Load from `appsettings.json` via `builder.Configuration.GetSection("Cors:AllowedOrigins").Get<string[]>()`.

**Dynamic origins:** `policy.SetIsOriginAllowed(origin => new Uri(origin).Host.EndsWith(".example.com"))`.

**Exposed headers:** `WithExposedHeaders("X-Total-Count", "X-Page-Number")` for custom response headers.

## Pitfalls to Avoid

- `AllowAnyOrigin()` in production (security risk)
- `AllowAnyOrigin()` + `AllowCredentials()` together (throws exception)
- Placing `UseCors()` after `UseRouting()` without `UseAuthorization()` between
- Multiple `.RequireCors()` on same endpoint (conflicts)
- Forgetting subdomains must be listed explicitly
