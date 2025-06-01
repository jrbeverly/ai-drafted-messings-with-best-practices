# Authentication

Secure user authentication. JWT tokens, OAuth 2.0, API keys. Protect endpoints.

## Principle

Authenticate all requests. Use strong tokens. Secure token storage. Short expiration with refresh.

## JWT Authentication

Configure JWT in ASP.NET Core:

```csharp
// Program.cs
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// JWT configuration
var jwtSettings = builder.Configuration.GetSection("Jwt");
var secretKey = jwtSettings["SecretKey"] ?? throw new InvalidOperationException("JWT secret key not configured");
var key = Encoding.UTF8.GetBytes(secretKey);

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.RequireHttpsMetadata = true;
    options.SaveToken = true;
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = jwtSettings["Issuer"],
        ValidAudience = jwtSettings["Audience"],
        IssuerSigningKey = new SymmetricSecurityKey(key),
        ClockSkew = TimeSpan.Zero
    };

    // Custom events
    options.Events = new JwtBearerEvents
    {
        OnAuthenticationFailed = context =>
        {
            if (context.Exception is SecurityTokenExpiredException)
            {
                context.Response.Headers["Token-Expired"] = "true";
            }
            return Task.CompletedTask;
        }
    };
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.Run();
```

Configuration:

```json
// appsettings.json
{
  "Jwt": {
    "SecretKey": "your-secret-key-at-least-32-characters-long",
    "Issuer": "https://api.example.com",
    "Audience": "https://api.example.com",
    "ExpirationMinutes": 15,
    "RefreshTokenExpirationDays": 7
  }
}
```

## Generate JWT Token

Token generation service:

```csharp
public interface ITokenService
{
    string GenerateAccessToken(User user);
    string GenerateRefreshToken();
    ClaimsPrincipal? GetPrincipalFromToken(string token);
}

public class TokenService : ITokenService
{
    private readonly IConfiguration _configuration;

    public TokenService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public string GenerateAccessToken(User user)
    {
        var jwtSettings = _configuration.GetSection("Jwt");
        var secretKey = jwtSettings["SecretKey"] ?? throw new InvalidOperationException("JWT secret not configured");
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secretKey));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var claims = new[]
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Id),
            new Claim(JwtRegisteredClaimNames.Email, user.Email),
            new Claim(JwtRegisteredClaimNames.Name, user.Name),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim("role", user.Role)
        };

        var expirationMinutes = int.Parse(jwtSettings["ExpirationMinutes"] ?? "15");

        var token = new JwtSecurityToken(
            issuer: jwtSettings["Issuer"],
            audience: jwtSettings["Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(expirationMinutes),
            signingCredentials: credentials
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken()
    {
        var randomBytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }

    public ClaimsPrincipal? GetPrincipalFromToken(string token)
    {
        var jwtSettings = _configuration.GetSection("Jwt");
        var secretKey = jwtSettings["SecretKey"] ?? throw new InvalidOperationException("JWT secret not configured");
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secretKey));

        var tokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = false,  // Don't validate expiration for refresh
            ValidateIssuerSigningKey = true,
            ValidIssuer = jwtSettings["Issuer"],
            ValidAudience = jwtSettings["Audience"],
            IssuerSigningKey = key
        };

        var tokenHandler = new JwtSecurityTokenHandler();

        try
        {
            var principal = tokenHandler.ValidateToken(token, tokenValidationParameters, out var securityToken);

            if (securityToken is not JwtSecurityToken jwtSecurityToken ||
                !jwtSecurityToken.Header.Alg.Equals(SecurityAlgorithms.HmacSha256, StringComparison.InvariantCultureIgnoreCase))
            {
                return null;
            }

            return principal;
        }
        catch
        {
            return null;
        }
    }
}
```

## Login Endpoint

Authentication endpoint:

```csharp
public record LoginRequest
{
    [Required]
    [EmailAddress]
    public string Email { get; init; } = string.Empty;

    [Required]
    public string Password { get; init; } = string.Empty;
}

public record LoginResponse
{
    public required string AccessToken { get; init; }
    public required string RefreshToken { get; init; }
    public required DateTime ExpiresAt { get; init; }
    public required UserResponse User { get; init; }
}

public static class LoginRoute
{
    public static async Task<IResult> HandleAsync(
        LoginRequest request,
        IUserService userService,
        ITokenService tokenService,
        IRefreshTokenRepository refreshTokenRepository)
    {
        // Validate credentials
        var user = await userService.ValidateCredentialsAsync(request.Email, request.Password);

        if (user == null)
        {
            return Results.Problem(
                type: "https://api.example.com/errors/invalid-credentials",
                title: "Invalid credentials",
                status: 401,
                detail: "Email or password is incorrect");
        }

        // Generate tokens
        var accessToken = tokenService.GenerateAccessToken(user);
        var refreshToken = tokenService.GenerateRefreshToken();

        // Store refresh token
        await refreshTokenRepository.SaveAsync(new RefreshToken
        {
            Token = refreshToken,
            UserId = user.Id,
            ExpiresAt = DateTime.UtcNow.AddDays(7),
            CreatedAt = DateTime.UtcNow
        });

        return Results.Ok(new LoginResponse
        {
            AccessToken = accessToken,
            RefreshToken = refreshToken,
            ExpiresAt = DateTime.UtcNow.AddMinutes(15),
            User = new UserResponse
            {
                Id = user.Id,
                Email = user.Email,
                Name = user.Name,
                CreatedAt = user.CreatedAt
            }
        });
    }
}
```

## Refresh Token

Token refresh endpoint:

```csharp
public record RefreshTokenRequest
{
    [Required]
    public string RefreshToken { get; init; } = string.Empty;
}

public static class RefreshTokenRoute
{
    public static async Task<IResult> HandleAsync(
        RefreshTokenRequest request,
        ITokenService tokenService,
        IRefreshTokenRepository refreshTokenRepository,
        IUserService userService)
    {
        // Validate refresh token
        var storedToken = await refreshTokenRepository.GetByTokenAsync(request.RefreshToken);

        if (storedToken == null || storedToken.IsRevoked || storedToken.ExpiresAt < DateTime.UtcNow)
        {
            return Results.Problem(
                type: "https://api.example.com/errors/invalid-token",
                title: "Invalid refresh token",
                status: 401,
                detail: "Refresh token is invalid or expired");
        }

        // Get user
        var user = await userService.GetUserByIdAsync(storedToken.UserId);

        if (user == null)
        {
            return Results.Problem(
                type: "https://api.example.com/errors/user-not-found",
                title: "User not found",
                status: 404,
                detail: "User associated with token not found");
        }

        // Generate new tokens
        var accessToken = tokenService.GenerateAccessToken(user);
        var newRefreshToken = tokenService.GenerateRefreshToken();

        // Revoke old refresh token
        await refreshTokenRepository.RevokeAsync(storedToken.Token);

        // Store new refresh token
        await refreshTokenRepository.SaveAsync(new RefreshToken
        {
            Token = newRefreshToken,
            UserId = user.Id,
            ExpiresAt = DateTime.UtcNow.AddDays(7),
            CreatedAt = DateTime.UtcNow
        });

        return Results.Ok(new LoginResponse
        {
            AccessToken = accessToken,
            RefreshToken = newRefreshToken,
            ExpiresAt = DateTime.UtcNow.AddMinutes(15),
            User = new UserResponse
            {
                Id = user.Id,
                Email = user.Email,
                Name = user.Name,
                CreatedAt = user.CreatedAt
            }
        });
    }
}
```

## Protected Endpoints

Require authentication:

```csharp
// Require authentication
app.MapGet("/users/me", async (HttpContext context, IUserService userService) =>
{
    var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

    if (userId == null)
        return Results.Unauthorized();

    var user = await userService.GetUserByIdAsync(userId);

    if (user == null)
        return Results.NotFound();

    return Results.Ok(user);
})
.RequireAuthorization();

// Or use [Authorize] attribute
public static class GetCurrentUserRoute
{
    [Authorize]
    public static async Task<IResult> HandleAsync(
        HttpContext context,
        IUserService userService)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value
            ?? throw new UnauthorizedAccessException();

        var user = await userService.GetUserByIdAsync(userId);

        if (user == null)
            return Results.NotFound();

        return Results.Ok(user);
    }
}
```

## Password Hashing

Secure password storage:

```csharp
public interface IPasswordHasher
{
    string HashPassword(string password);
    bool VerifyPassword(string password, string passwordHash);
}

public class PasswordHasher : IPasswordHasher
{
    private const int SaltSize = 16;
    private const int HashSize = 32;
    private const int Iterations = 100000;

    public string HashPassword(string password)
    {
        // Generate salt
        var salt = RandomNumberGenerator.GetBytes(SaltSize);

        // Generate hash
        var hash = Rfc2898DeriveBytes.Pbkdf2(
            password,
            salt,
            Iterations,
            HashAlgorithmName.SHA256,
            HashSize);

        // Combine salt and hash
        var hashBytes = new byte[SaltSize + HashSize];
        Array.Copy(salt, 0, hashBytes, 0, SaltSize);
        Array.Copy(hash, 0, hashBytes, SaltSize, HashSize);

        return Convert.ToBase64String(hashBytes);
    }

    public bool VerifyPassword(string password, string passwordHash)
    {
        // Decode stored hash
        var hashBytes = Convert.FromBase64String(passwordHash);

        // Extract salt
        var salt = new byte[SaltSize];
        Array.Copy(hashBytes, 0, salt, 0, SaltSize);

        // Extract hash
        var storedHash = new byte[HashSize];
        Array.Copy(hashBytes, SaltSize, storedHash, 0, HashSize);

        // Compute hash from input password
        var computedHash = Rfc2898DeriveBytes.Pbkdf2(
            password,
            salt,
            Iterations,
            HashAlgorithmName.SHA256,
            HashSize);

        // Compare hashes (constant-time comparison)
        return CryptographicOperations.FixedTimeEquals(storedHash, computedHash);
    }
}

// Usage
public async Task<User> CreateUserAsync(CreateUserRequest request)
{
    var passwordHash = _passwordHasher.HashPassword(request.Password);

    var user = new User
    {
        Id = Guid.NewGuid().ToString(),
        Email = request.Email,
        Name = request.Name,
        PasswordHash = passwordHash,
        CreatedAt = DateTime.UtcNow
    };

    await _userRepository.SaveAsync(user);
    return user;
}
```

## API Key Authentication

API key support:

```csharp
// API Key authentication handler
public class ApiKeyAuthenticationHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    private const string ApiKeyHeaderName = "X-API-Key";
    private readonly IApiKeyService _apiKeyService;

    public ApiKeyAuthenticationHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder,
        IApiKeyService apiKeyService)
        : base(options, logger, encoder)
    {
        _apiKeyService = apiKeyService;
    }

    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue(ApiKeyHeaderName, out var apiKeyValues))
        {
            return AuthenticateResult.Fail("API Key not found");
        }

        var apiKey = apiKeyValues.FirstOrDefault();

        if (string.IsNullOrEmpty(apiKey))
        {
            return AuthenticateResult.Fail("API Key is empty");
        }

        var apiKeyInfo = await _apiKeyService.ValidateAsync(apiKey);

        if (apiKeyInfo == null)
        {
            return AuthenticateResult.Fail("Invalid API Key");
        }

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, apiKeyInfo.UserId),
            new Claim(ClaimTypes.Name, apiKeyInfo.Name),
            new Claim("api_key_id", apiKeyInfo.Id)
        };

        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);

        return AuthenticateResult.Success(ticket);
    }
}

// Register
builder.Services.AddAuthentication("ApiKey")
    .AddScheme<AuthenticationSchemeOptions, ApiKeyAuthenticationHandler>("ApiKey", null);

// Use on endpoint
app.MapGet("/data", GetData)
    .RequireAuthorization(new AuthorizeAttribute { AuthenticationSchemes = "ApiKey" });
```

## OAuth 2.0 Integration

External authentication providers:

```csharp
// Add OAuth providers
builder.Services.AddAuthentication()
    .AddGoogle(options =>
    {
        options.ClientId = builder.Configuration["Authentication:Google:ClientId"];
        options.ClientSecret = builder.Configuration["Authentication:Google:ClientSecret"];
        options.CallbackPath = "/signin-google";
    })
    .AddGitHub(options =>
    {
        options.ClientId = builder.Configuration["Authentication:GitHub:ClientId"];
        options.ClientSecret = builder.Configuration["Authentication:GitHub:ClientSecret"];
        options.CallbackPath = "/signin-github";
    });

// OAuth callback
app.MapGet("/auth/{provider}/callback", async (
    string provider,
    HttpContext context,
    ITokenService tokenService,
    IUserService userService) =>
{
    // Authenticate with external provider
    var result = await context.AuthenticateAsync(provider);

    if (!result.Succeeded)
    {
        return Results.Redirect("/login?error=authentication_failed");
    }

    var email = result.Principal.FindFirst(ClaimTypes.Email)?.Value;
    var name = result.Principal.FindFirst(ClaimTypes.Name)?.Value;

    if (email == null)
    {
        return Results.Redirect("/login?error=email_required");
    }

    // Get or create user
    var user = await userService.GetOrCreateFromOAuthAsync(email, name, provider);

    // Generate JWT tokens
    var accessToken = tokenService.GenerateAccessToken(user);
    var refreshToken = tokenService.GenerateRefreshToken();

    return Results.Redirect($"/auth/success?token={accessToken}&refresh={refreshToken}");
});
```

## Logout

Revoke tokens:

```csharp
public static class LogoutRoute
{
    public static async Task<IResult> HandleAsync(
        HttpContext context,
        IRefreshTokenRepository refreshTokenRepository)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        if (userId == null)
            return Results.Unauthorized();

        // Revoke all refresh tokens for user
        await refreshTokenRepository.RevokeAllForUserAsync(userId);

        return Results.NoContent();
    }
}

// Client must discard access token
```

## Guidelines

**Token Security:**
- Use strong secret keys (minimum 256 bits)
- Short access token lifetime (15 minutes)
- Longer refresh token lifetime (7 days)
- Revoke refresh tokens on logout
- HTTPS only in production

**Password Security:**
- Hash with PBKDF2, bcrypt, or Argon2
- Minimum 100,000 iterations
- Unique salt per password
- Never store plain text passwords
- Enforce strong password policies

**API Keys:**
- Long random values (minimum 256 bits)
- Store hashed, not plain text
- Allow revocation
- Scope permissions per key
- Rotate regularly

**Authentication:**
- Validate all inputs
- Constant-time comparisons
- Rate limit login attempts
- Log authentication failures
- Implement account lockout

**Token Validation:**
- Validate issuer and audience
- Check expiration
- Verify signature
- Use secure algorithms (HS256, RS256)
- No client-side token validation

## Benefits

Secure. Industry-standard JWT authentication.

Stateless. No server-side session storage.

Scalable. Tokens validated independently.

Flexible. Multiple authentication methods.

## Related

- [authorization.md](./authorization.md) - Permission management
- [secrets-management.md](./secrets-management.md) - Secure configuration
- [owasp-security.md](./owasp-security.md) - Security best practices
