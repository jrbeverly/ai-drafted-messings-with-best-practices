# JWT Authentication

Stateless, token-based authentication for APIs using JSON Web Tokens with refresh token support.

## Why It Matters

- No server-side session storage -- scales horizontally across servers
- Industry-standard format understood by all platforms and languages
- Claims embedded in the token enable stateless authorization decisions

## Key Recommendations

- **Configuration**: Validate issuer, audience, signing key, and lifetime on every request

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true, ValidateAudience = true,
            ValidateIssuerSigningKey = true, ValidateLifetime = true,
            ValidIssuer = config["Jwt:Issuer"], ValidAudience = config["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(config["Jwt:SecretKey"]!)),
            ClockSkew = TimeSpan.Zero
        };
    });
```

- **Access tokens**: Short-lived (15 minutes), include `sub`, `email`, `role`, `jti` claims
- **Refresh tokens**: Cryptographically random (64 bytes), stored hashed, 7-day expiry, one per user
- **Token generation**: Use `JwtSecurityTokenHandler`, sign with `HmacSha256`, include `jti` for uniqueness
- **Refresh flow**: Validate expired access token (skip lifetime check), verify refresh token matches stored, rotate both
- **Logout**: Revoke refresh token (set to null); access token expires naturally
- **Password hashing**: PBKDF2 with 100k iterations, 16-byte salt, `CryptographicOperations.FixedTimeEquals`
- **Middleware order**: `app.UseAuthentication()` before `app.UseAuthorization()`

## Endpoint Pattern

```csharp
app.MapPost("/auth/login", async (LoginRequest req, IUserService users, ITokenService tokens) =>
{
    var user = await users.AuthenticateAsync(req.Email, req.Password);
    if (user is null) return Results.Problem(status: 401, title: "Invalid Credentials");
    var access = tokens.GenerateAccessToken(user);
    var refresh = tokens.GenerateRefreshToken();
    await users.SaveRefreshTokenAsync(user.Id, refresh);
    return Results.Ok(new { access, refresh, expiresAt = DateTime.UtcNow.AddMinutes(15) });
}).AllowAnonymous();
```

## Pitfalls to Avoid

- Storing the secret key in `appsettings.json` (use AWS Secrets Manager or environment variables)
- Allowing default `ClockSkew` (5 min) -- set to `TimeSpan.Zero`
- Including sensitive data in claims (SSN, credit card numbers)
- Not revoking refresh tokens on logout
- Returning specific "user not found" vs "wrong password" errors (reveals user existence)
- Skipping algorithm validation in `GetPrincipalFromExpiredToken`
