# API Key Authentication

Simple header-based authentication for server-to-server communication, webhooks, and public APIs.

## Why It Matters

- Stateless and easy to implement for machine-to-machine auth
- Supports scoped permissions and expiration for fine-grained access control
- Auditable -- each key is tied to a user/service for usage tracking

## Key Recommendations

- **Header transport**: Use `X-API-Key` header (never query parameters -- they get logged/cached)
- **Key generation**: Cryptographically random, 32+ bytes, prefixed for identification (`sk_...`)
- **Storage**: Hash keys before persisting (like passwords); return plain key only once at creation
- **Validation middleware**: Check key on every request, skip public endpoints explicitly

```csharp
// Combine JWT + API key auth schemes
builder.Services.AddAuthentication()
    .AddJwtBearer("Bearer", options => { /* ... */ })
    .AddScheme<AuthenticationSchemeOptions, ApiKeyAuthenticationHandler>("ApiKey", _ => { });

builder.Services.AddAuthorization(options =>
{
    options.DefaultPolicy = new AuthorizationPolicyBuilder()
        .AddAuthenticationSchemes("Bearer", "ApiKey")
        .RequireAuthenticatedUser()
        .Build();
});
```

- **Scope-based authorization**: Attach scopes to keys, enforce via `IAuthorizationHandler`
- **Rate limiting**: Rate limit per API key using `IMemoryCache` or built-in `AddRateLimiter`
- **Key lifecycle**: Support revocation, expiration (`ExpiresAt`), and rotation
- **HTTPS only**: Never transmit keys over unencrypted connections

## Pitfalls to Avoid

- Storing keys in plaintext in the database
- Returning the key on subsequent GET requests (only at creation)
- Putting keys in URL query strings (visible in logs, browser history, proxies)
- Missing rate limiting -- a leaked key without limits is an open door
- Forgetting to check `IsActive` and `ExpiresAt` during validation
- Using `string ==` instead of constant-time comparison for key validation
