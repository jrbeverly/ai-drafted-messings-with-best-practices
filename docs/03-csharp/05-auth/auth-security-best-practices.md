# Authentication & Authorization Security Best Practices

Cross-cutting security guidelines for auth systems: defense in depth, secure defaults, and audit everything.

## Why It Matters

- Auth is the primary attack surface -- weak auth means full compromise
- Industry compliance (OWASP, SOC2) requires specific security controls
- Proper logging and monitoring catch breaches early

## Password Security

- **Hashing**: PBKDF2 (100k+ iterations), bcrypt, or Argon2 -- never MD5/SHA alone
- **Comparison**: Always `CryptographicOperations.FixedTimeEquals` (prevents timing attacks)
- **Policy**: Minimum 12 characters, check against common password lists
- **Account lockout**: Lock after 5 failed attempts for 15 minutes; notify user

## Token Security

- **Short-lived access tokens**: 15 minutes max expiration, `ClockSkew = TimeSpan.Zero`
- **Refresh token rotation**: Issue new refresh token on each use; detect reuse as theft
- **Validate on every request**: Check user still exists and is active in `OnTokenValidated`
- **Store secrets in AWS Secrets Manager** (not `appsettings.json`)

```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidateIssuer = true, ValidateAudience = true,
    ValidateIssuerSigningKey = true, ValidateLifetime = true,
    ClockSkew = TimeSpan.Zero, RequireExpirationTime = true
};
```

## Rate Limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("auth", o => { o.Window = TimeSpan.FromMinutes(1); o.PermitLimit = 5; });
    options.AddFixedWindowLimiter("api", o => { o.Window = TimeSpan.FromMinutes(1); o.PermitLimit = 100; });
});
```

## Security Headers

- `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`
- `Strict-Transport-Security` with `includeSubDomains; preload`
- CORS: Explicit origins only -- never `AllowAnyOrigin()` in production

## Audit Logging

- Log every login attempt (success and failure), token refresh, and authorization denial
- Include timestamp, user ID, IP address, and event type
- Alert on suspicious patterns (burst failures, token reuse)

## Sensitive Data Protection

- `[JsonIgnore]` on password fields to prevent serialization in logs
- Never expose user existence in error messages ("Email or password is incorrect")
- Remove `Server` header from responses

## Pitfalls to Avoid

- Using `AllowAnyOrigin()` in CORS configuration
- Generic "An error occurred" messages that don't help debugging but also leak nothing useful
- Skipping audit logging for auth events
- Allowing `ClockSkew` defaults (5 minutes) -- use `TimeSpan.Zero`
- Not revoking all tokens on suspected refresh token theft
