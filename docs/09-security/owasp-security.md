# OWASP Security Best Practices

Protection against OWASP Top 10 vulnerabilities. Secure coding practices. Input validation. Output encoding.

## Principle

Defense in depth. Validate input. Encode output. Principle of least privilege. Fail securely.

## Injection Prevention

SQL Injection (use parameterized queries):

```csharp
// GOOD - Parameterized query (DynamoDB)
var request = new QueryRequest
{
    TableName = tableName,
    KeyConditionExpression = "PK = :pk",
    ExpressionAttributeValues = new Dictionary<string, AttributeValue>
    {
        [":pk"] = new AttributeValue { S = userId }  // Safe
    }
};

// BAD - String concatenation (vulnerable to injection)
var request = new QueryRequest
{
    KeyConditionExpression = $"PK = {userId}"  // DANGEROUS!
};
```

NoSQL Injection prevention:

```csharp
// Validate input
public async Task<User?> GetUserAsync(string userId)
{
    // Validate format
    if (!Regex.IsMatch(userId, @"^[a-zA-Z0-9_-]+$"))
    {
        throw new ValidationException("Invalid user ID format");
    }

    // Use parameterized queries
    var request = new GetItemRequest
    {
        TableName = tableName,
        Key = new Dictionary<string, AttributeValue>
        {
            ["PK"] = new AttributeValue { S = $"USER#{userId}" }
        }
    };

    var response = await dynamoDb.GetItemAsync(request);
    return MapToUser(response.Item);
}
```

## XSS Prevention

Output encoding:

```csharp
// ASP.NET Core automatically encodes output
// Razor views, minimal API JSON responses are safe by default

// Manual encoding when needed
using System.Web;

public string SanitizeHtml(string input)
{
    return HttpUtility.HtmlEncode(input);
}

// Content Security Policy
app.Use(async (context, next) =>
{
    context.Response.Headers.Add(
        "Content-Security-Policy",
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:;");

    await next();
});
```

Input validation:

```csharp
public record CreatePostRequest
{
    [Required]
    [StringLength(200, MinimumLength = 1)]
    [RegularExpression(@"^[a-zA-Z0-9\s.,!?'-]+$", ErrorMessage = "Invalid characters in title")]
    public string Title { get; init; } = string.Empty;

    [Required]
    [StringLength(5000)]
    public string Content { get; init; } = string.Empty;
}

// Sanitize HTML if allowing rich text
public string SanitizeRichText(string html)
{
    // Use library like HtmlSanitizer
    var sanitizer = new HtmlSanitizer();
    sanitizer.AllowedTags.Add("p");
    sanitizer.AllowedTags.Add("b");
    sanitizer.AllowedTags.Add("i");
    sanitizer.AllowedTags.Add("strong");
    sanitizer.AllowedTags.Add("em");

    return sanitizer.Sanitize(html);
}
```

## CSRF Protection

Anti-forgery tokens:

```csharp
// Enable anti-forgery
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "X-XSRF-TOKEN";
    options.Cookie.Name = "XSRF-TOKEN";
    options.Cookie.SameSite = SameSiteMode.Strict;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});

// Require anti-forgery token
app.MapPost("/users", CreateUser)
    .RequireAntiforgery();

// For API: use SameSite cookies + custom headers
app.Use(async (context, next) =>
{
    if (context.Request.Method != "GET" &&
        context.Request.Method != "HEAD" &&
        context.Request.Method != "OPTIONS")
    {
        // Require custom header for state-changing operations
        if (!context.Request.Headers.ContainsKey("X-Requested-With"))
        {
            context.Response.StatusCode = 400;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Missing required header: X-Requested-With"
            });
            return;
        }
    }

    await next();
});
```

## Authentication Security

Secure password storage:

```csharp
// Use strong hashing (PBKDF2, bcrypt, or Argon2)
public class PasswordHasher
{
    private const int Iterations = 100000;  // OWASP recommendation

    public string HashPassword(string password)
    {
        var salt = RandomNumberGenerator.GetBytes(16);

        var hash = Rfc2898DeriveBytes.Pbkdf2(
            password,
            salt,
            Iterations,
            HashAlgorithmName.SHA256,
            32);

        var hashBytes = new byte[48];
        Array.Copy(salt, 0, hashBytes, 0, 16);
        Array.Copy(hash, 0, hashBytes, 16, 32);

        return Convert.ToBase64String(hashBytes);
    }

    public bool VerifyPassword(string password, string passwordHash)
    {
        var hashBytes = Convert.FromBase64String(passwordHash);

        var salt = new byte[16];
        Array.Copy(hashBytes, 0, salt, 0, 16);

        var storedHash = new byte[32];
        Array.Copy(hashBytes, 16, storedHash, 0, 32);

        var computedHash = Rfc2898DeriveBytes.Pbkdf2(
            password,
            salt,
            Iterations,
            HashAlgorithmName.SHA256,
            32);

        return CryptographicOperations.FixedTimeEquals(storedHash, computedHash);
    }
}
```

Account lockout:

```csharp
public class LoginAttemptService
{
    private readonly IMemoryCache _cache;
    private const int MaxAttempts = 5;
    private static readonly TimeSpan LockoutDuration = TimeSpan.FromMinutes(15);

    public async Task<bool> IsLockedOutAsync(string email)
    {
        var key = $"lockout:{email}";
        return _cache.TryGetValue(key, out _);
    }

    public async Task RecordFailedAttemptAsync(string email)
    {
        var key = $"attempts:{email}";

        var attempts = _cache.GetOrCreate(key, entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15);
            return 0;
        });

        attempts++;

        if (attempts >= MaxAttempts)
        {
            // Lock account
            _cache.Set($"lockout:{email}", true, LockoutDuration);

            // Clear attempts
            _cache.Remove(key);
        }
        else
        {
            _cache.Set(key, attempts, TimeSpan.FromMinutes(15));
        }
    }

    public async Task ClearAttemptsAsync(string email)
    {
        var key = $"attempts:{email}";
        _cache.Remove(key);
    }
}
```

## Sensitive Data Exposure

Secure headers:

```csharp
app.Use(async (context, next) =>
{
    // Security headers
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");
    context.Response.Headers.Add("Permissions-Policy", "geolocation=(), microphone=(), camera=()");

    // HSTS (HTTPS only)
    if (context.Request.IsHttps)
    {
        context.Response.Headers.Add(
            "Strict-Transport-Security",
            "max-age=31536000; includeSubDomains; preload");
    }

    await next();
});
```

Remove sensitive data from logs:

```csharp
public class SensitiveDataFilter : ILogger
{
    private readonly ILogger _inner;
    private static readonly Regex PasswordPattern = new(@"password[""']?\s*[:=]\s*[""']?([^""'\s,}]+)", RegexOptions.IgnoreCase);
    private static readonly Regex TokenPattern = new(@"(token|key|secret)[""']?\s*[:=]\s*[""']?([^""'\s,}]+)", RegexOptions.IgnoreCase);

    public void Log<TState>(
        LogLevel logLevel,
        EventId eventId,
        TState state,
        Exception? exception,
        Func<TState, Exception?, string> formatter)
    {
        var message = formatter(state, exception);

        // Redact sensitive data
        message = PasswordPattern.Replace(message, "password=***REDACTED***");
        message = TokenPattern.Replace(message, "$1=***REDACTED***");

        _inner.Log(logLevel, eventId, message, exception, (s, e) => message);
    }
}
```

## Broken Access Control

Resource-level authorization:

```csharp
public static async Task<IResult> GetLoan(
    string id,
    HttpContext context,
    ILoanService loanService)
{
    var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value
        ?? throw new UnauthorizedAccessException();

    var loan = await loanService.GetLoanByIdAsync(id);

    if (loan == null)
        return Results.NotFound();

    // Check ownership
    if (loan.UserId != userId && !context.User.IsInRole(Roles.Admin))
    {
        return Results.Problem(
            type: "https://api.example.com/errors/forbidden",
            title: "Forbidden",
            status: 403,
            detail: "You do not have permission to access this resource");
    }

    return Results.Ok(loan);
}
```

Direct object reference protection:

```csharp
// BAD - Predictable IDs
public string GenerateId()
{
    return _counter++.ToString();  // Sequential: 1, 2, 3, ...
}

// GOOD - Random IDs
public string GenerateId()
{
    return $"loan_{Guid.NewGuid():N}";  // Random: loan_a1b2c3d4...
}

// GOOD - Encrypted IDs
public string EncryptId(string id)
{
    // Encrypt ID before exposing to client
    using var aes = Aes.Create();
    aes.Key = _encryptionKey;
    aes.GenerateIV();

    var encryptor = aes.CreateEncryptor();
    var plainBytes = Encoding.UTF8.GetBytes(id);
    var cipherBytes = encryptor.TransformFinalBlock(plainBytes, 0, plainBytes.Length);

    var result = new byte[aes.IV.Length + cipherBytes.Length];
    Array.Copy(aes.IV, result, aes.IV.Length);
    Array.Copy(cipherBytes, 0, result, aes.IV.Length, cipherBytes.Length);

    return Convert.ToBase64String(result);
}
```

## Security Misconfiguration

Disable default accounts:

```csharp
// Remove default Swagger in production
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Disable detailed error messages in production
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error");
}
else
{
    app.UseDeveloperExceptionPage();
}
```

Secure configuration:

```csharp
// Remove server header
app.Use(async (context, next) =>
{
    context.Response.Headers.Remove("Server");
    await next();
});

// Configure Kestrel
builder.WebHost.ConfigureKestrel(options =>
{
    options.AddServerHeader = false;
    options.Limits.MaxRequestBodySize = 10 * 1024 * 1024;  // 10 MB
    options.Limits.MaxRequestLineSize = 8192;
});
```

## Vulnerable Components

Dependency scanning:

```bash
# Check for vulnerabilities
dotnet list package --vulnerable

# Update packages
dotnet outdated

# Restore with audit
dotnet restore --audit
```

GitHub Dependabot:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

## Rate Limiting

Prevent abuse:

```csharp
builder.Services.AddRateLimiter(options =>
{
    // Fixed window
    options.AddFixedWindowLimiter("fixed", options =>
    {
        options.Window = TimeSpan.FromMinutes(1);
        options.PermitLimit = 100;
        options.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        options.QueueLimit = 0;
    });

    // Sliding window
    options.AddSlidingWindowLimiter("sliding", options =>
    {
        options.Window = TimeSpan.FromMinutes(1);
        options.PermitLimit = 100;
        options.SegmentsPerWindow = 6;
    });

    // Token bucket
    options.AddTokenBucketLimiter("token", options =>
    {
        options.TokenLimit = 100;
        options.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        options.TokensPerPeriod = 10;
    });
});

app.UseRateLimiter();

// Apply to endpoint
app.MapGet("/users", GetUsers)
    .RequireRateLimiting("fixed");
```

## Request Validation

Input size limits:

```csharp
// Request body size limit
builder.Services.Configure<FormOptions>(options =>
{
    options.ValueLengthLimit = int.MaxValue;
    options.MultipartBodyLengthLimit = 10 * 1024 * 1024;  // 10 MB
});

// Per-endpoint limit
app.MapPost("/upload", UploadFile)
    .DisableRequestSizeLimit();  // Only when necessary
```

Path traversal prevention:

```csharp
public async Task<IResult> GetFile(string filename)
{
    // Validate filename
    if (filename.Contains("..") || Path.IsPathRooted(filename))
    {
        return Results.BadRequest(new { error = "Invalid filename" });
    }

    var basePath = Path.Combine(Directory.GetCurrentDirectory(), "uploads");
    var fullPath = Path.Combine(basePath, filename);

    // Ensure file is within allowed directory
    if (!fullPath.StartsWith(basePath))
    {
        return Results.BadRequest(new { error = "Invalid file path" });
    }

    if (!File.Exists(fullPath))
    {
        return Results.NotFound();
    }

    var bytes = await File.ReadAllBytesAsync(fullPath);
    return Results.File(bytes, "application/octet-stream", filename);
}
```

## Security Testing

Automated security tests:

```csharp
public class SecurityTests
{
    [Fact]
    public async Task Endpoint_WithoutAuth_Returns401()
    {
        var client = _factory.CreateClient();

        var response = await client.GetAsync("/users/me");

        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }

    [Fact]
    public async Task Endpoint_WithInvalidToken_Returns401()
    {
        var client = _factory.CreateClient();
        client.DefaultRequestHeaders.Add("Authorization", "Bearer invalid-token");

        var response = await client.GetAsync("/users/me");

        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }

    [Fact]
    public async Task Endpoint_UserAccessingOthersResource_Returns403()
    {
        var client = _factory.CreateClient();
        client.DefaultRequestHeaders.Add("Authorization", $"Bearer {GetUserToken()}");

        // Try to access another user's resource
        var response = await client.GetAsync("/users/other_user_id");

        Assert.Equal(HttpStatusCode.Forbidden, response.StatusCode);
    }
}
```

## Guidelines

**Input Validation:**
- Validate all input (whitelisting preferred)
- Use strong typing and validation attributes
- Sanitize before storage
- Encode before output

**Authentication:**
- Strong password hashing (PBKDF2, bcrypt, Argon2)
- Multi-factor authentication
- Account lockout after failed attempts
- Secure session management

**Authorization:**
- Enforce on every request
- Resource-level checks
- Deny by default
- Audit authorization failures

**Encryption:**
- HTTPS everywhere (TLS 1.2+)
- Encrypt sensitive data at rest
- Use strong encryption algorithms
- Secure key management

**Error Handling:**
- Generic error messages to users
- Detailed logging for developers
- Never expose stack traces
- Monitor error patterns

## Benefits

Secure. Protection against common vulnerabilities.

Compliant. Meets security standards.

Auditable. Security events logged.

Resilient. Defense in depth.

## Related

- [authentication.md](./authentication.md) - User authentication
- [authorization.md](./authorization.md) - Permission management
- [secrets-management.md](./secrets-management.md) - Secure configuration
- [api-error-handling.md](../08-api-design/api-error-handling.md) - Error handling
