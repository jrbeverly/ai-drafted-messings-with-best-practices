# MIME Type Versioning

Version APIs using media types (Content-Type/Accept). Vendor-specific formats. Backwards compatibility.

## Principle

Use Accept/Content-Type headers for API versioning. Cleaner URLs. Client-driven version selection.

## Basic MIME Type Versioning

```csharp
app.MapGet("/users", (HttpContext context) =>
{
    var acceptHeader = context.Request.Headers.Accept.ToString();

    // application/vnd.myapp.v1+json
    if (acceptHeader.Contains("vnd.myapp.v1+json"))
    {
        return Results.Json(new { Version = "v1", Users = GetUsersV1() });
    }

    // application/vnd.myapp.v2+json
    if (acceptHeader.Contains("vnd.myapp.v2+json"))
    {
        return Results.Json(new { Version = "v2", Users = GetUsersV2() });
    }

    // Default to latest version
    return Results.Json(new { Version = "v2", Users = GetUsersV2() });
});

// Request v1:
// GET /users
// Accept: application/vnd.myapp.v1+json

// Request v2:
// GET /users
// Accept: application/vnd.myapp.v2+json
```

## Vendor-Specific Media Types

```csharp
// Define custom media types
public static class MediaTypes
{
    // v1 formats
    public const string UserV1Json = "application/vnd.myapp.user.v1+json";
    public const string UserV1Xml = "application/vnd.myapp.user.v1+xml";

    // v2 formats
    public const string UserV2Json = "application/vnd.myapp.user.v2+json";
    public const string UserV2Xml = "application/vnd.myapp.user.v2+xml";

    // Latest (no version)
    public const string UserJson = "application/vnd.myapp.user+json";
}

app.MapGet("/users/{id}", (HttpContext context, string id) =>
{
    var accept = context.Request.Headers.Accept.ToString();

    // Match specific versions
    if (accept.Contains(MediaTypes.UserV1Json))
    {
        var user = GetUserV1(id);
        return Results.Json(user);
    }

    if (accept.Contains(MediaTypes.UserV2Json))
    {
        var user = GetUserV2(id);
        return Results.Json(user);
    }

    // Default to latest
    var latestUser = GetUserV2(id);
    return Results.Json(latestUser);
});
```

## Media Type Negotiation Middleware

```csharp
public class MediaTypeVersioningMiddleware
{
    private readonly RequestDelegate _next;

    public MediaTypeVersioningMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var accept = context.Request.Headers.Accept.ToString();

        // Parse version from Accept header
        var version = ExtractVersion(accept);

        // Store in HttpContext for later use
        context.Items["ApiVersion"] = version;

        await _next(context);
    }

    private static string ExtractVersion(string acceptHeader)
    {
        // application/vnd.myapp.v1+json -> "v1"
        // application/vnd.myapp.v2+json -> "v2"
        var match = Regex.Match(acceptHeader, @"vnd\.myapp\.v(\d+)\+");

        return match.Success ? $"v{match.Groups[1].Value}" : "v2"; // Default to v2
    }
}

app.UseMiddleware<MediaTypeVersioningMiddleware>();

// Use version in endpoint
app.MapGet("/users", (HttpContext context) =>
{
    var version = context.Items["ApiVersion"]?.ToString() ?? "v2";

    return version switch
    {
        "v1" => Results.Json(GetUsersV1()),
        "v2" => Results.Json(GetUsersV2()),
        _ => Results.Json(GetUsersV2())
    };
});
```

## Version-Specific Response Models

```csharp
// v1 model
public record UserV1
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
}

// v2 model (more detailed)
public record UserV2
{
    public string Id { get; init; } = "";  // Changed from int to string
    public string FirstName { get; init; } = "";  // Split name
    public string LastName { get; init; } = "";
    public string Email { get; init; } = "";
    public DateTime CreatedAt { get; init; }
}

app.MapGet("/users/{id}", (string id, HttpContext context, IUserService userService) =>
{
    var accept = context.Request.Headers.Accept.ToString();

    var user = userService.GetUser(id);

    if (user is null)
    {
        return Results.NotFound();
    }

    // Return v1 format
    if (accept.Contains("v1"))
    {
        var userV1 = new UserV1
        {
            Id = int.Parse(user.Id),
            Name = $"{user.FirstName} {user.LastName}"
        };

        return Results.Json(userV1);
    }

    // Return v2 format
    var userV2 = new UserV2
    {
        Id = user.Id,
        FirstName = user.FirstName,
        LastName = user.LastName,
        Email = user.Email,
        CreatedAt = user.CreatedAt
    };

    return Results.Json(userV2);
});
```

## Content-Type Based Routing

```csharp
// Different endpoints based on Content-Type
app.MapPost("/users", async (HttpContext context, IUserService userService) =>
{
    var contentType = context.Request.ContentType;

    if (contentType?.Contains("vnd.myapp.v1+json") == true)
    {
        var requestV1 = await context.Request.ReadFromJsonAsync<CreateUserRequestV1>();
        var user = await userService.CreateUserV1Async(requestV1!);
        return Results.Created($"/users/{user.Id}", user);
    }

    if (contentType?.Contains("vnd.myapp.v2+json") == true)
    {
        var requestV2 = await context.Request.ReadFromJsonAsync<CreateUserRequestV2>();
        var user = await userService.CreateUserV2Async(requestV2!);
        return Results.Created($"/users/{user.Id}", user);
    }

    return Results.BadRequest(new
    {
        Error = "Unsupported Content-Type",
        SupportedTypes = new[]
        {
            "application/vnd.myapp.v1+json",
            "application/vnd.myapp.v2+json"
        }
    });
});
```

## Version Negotiation with Quality Values

```csharp
app.MapGet("/users", (HttpContext context) =>
{
    var accept = context.Request.Headers.Accept.ToString();

    // Accept: application/vnd.myapp.v2+json;q=1.0, application/vnd.myapp.v1+json;q=0.5
    var preferences = ParseAcceptHeader(accept);

    // Get highest quality version
    var preferredVersion = preferences.OrderByDescending(p => p.Quality).First();

    return preferredVersion.Version switch
    {
        "v1" => Results.Json(GetUsersV1()),
        "v2" => Results.Json(GetUsersV2()),
        _ => Results.Json(GetUsersV2())
    };
});

static List<(string Version, double Quality)> ParseAcceptHeader(string accept)
{
    var result = new List<(string Version, double Quality)>();

    foreach (var part in accept.Split(','))
    {
        var segments = part.Trim().Split(';');
        var mediaType = segments[0];

        // Extract version
        var versionMatch = Regex.Match(mediaType, @"\.v(\d+)\+");
        if (!versionMatch.Success) continue;

        var version = $"v{versionMatch.Groups[1].Value}";

        // Extract quality
        var quality = 1.0;
        if (segments.Length > 1)
        {
            var qMatch = Regex.Match(segments[1], @"q=([0-9.]+)");
            if (qMatch.Success)
            {
                double.TryParse(qMatch.Groups[1].Value, out quality);
            }
        }

        result.Add((version, quality));
    }

    return result;
}
```

## Deprecation Warnings

```csharp
app.MapGet("/users", (HttpContext context) =>
{
    var accept = context.Request.Headers.Accept.ToString();

    if (accept.Contains("v1"))
    {
        // Add deprecation warning
        context.Response.Headers.Add("Warning",
            "299 - \"API version v1 is deprecated. Please upgrade to v2. " +
            "v1 will be removed on 2026-12-31.\"");

        context.Response.Headers.Add("Sunset",
            "Sat, 31 Dec 2026 23:59:59 GMT");

        context.Response.Headers.Add("Link",
            "</docs/migration-v1-to-v2>; rel=\"deprecation\"");

        return Results.Json(GetUsersV1());
    }

    return Results.Json(GetUsersV2());
});
```

## Combine with URL Versioning

```csharp
// Support both URL and media type versioning
app.MapGet("/api/v1/users", (HttpContext context) =>
{
    // URL version = v1, always return v1
    return Results.Json(GetUsersV1());
});

app.MapGet("/api/v2/users", (HttpContext context) =>
{
    // URL version = v2, check media type for sub-versions
    var accept = context.Request.Headers.Accept.ToString();

    if (accept.Contains("vnd.myapp.v2.1"))
    {
        return Results.Json(GetUsersV2_1());  // Minor version
    }

    return Results.Json(GetUsersV2());
});

app.MapGet("/api/users", (HttpContext context) =>
{
    // No URL version, use media type versioning
    var accept = context.Request.Headers.Accept.ToString();

    if (accept.Contains("v1"))
        return Results.Json(GetUsersV1());

    if (accept.Contains("v2"))
        return Results.Json(GetUsersV2());

    // Default to latest
    return Results.Json(GetUsersV2());
});
```

## Strongly-Typed Media Type Handler

```csharp
public interface IMediaTypeHandler
{
    bool CanHandle(string mediaType);
    Task<IResult> HandleAsync(HttpContext context);
}

public class UserV1Handler : IMediaTypeHandler
{
    private readonly IUserService _userService;

    public UserV1Handler(IUserService userService)
    {
        _userService = userService;
    }

    public bool CanHandle(string mediaType)
    {
        return mediaType.Contains("vnd.myapp.user.v1");
    }

    public async Task<IResult> HandleAsync(HttpContext context)
    {
        var id = context.Request.RouteValues["id"]?.ToString();
        var user = await _userService.GetUserAsync(id!);

        if (user is null)
            return Results.NotFound();

        var userV1 = new UserV1
        {
            Id = int.Parse(user.Id),
            Name = $"{user.FirstName} {user.LastName}"
        };

        return Results.Json(userV1);
    }
}

public class UserV2Handler : IMediaTypeHandler
{
    private readonly IUserService _userService;

    public UserV2Handler(IUserService userService)
    {
        _userService = userService;
    }

    public bool CanHandle(string mediaType)
    {
        return mediaType.Contains("vnd.myapp.user.v2");
    }

    public async Task<IResult> HandleAsync(HttpContext context)
    {
        var id = context.Request.RouteValues["id"]?.ToString();
        var user = await _userService.GetUserAsync(id!);

        if (user is null)
            return Results.NotFound();

        var userV2 = new UserV2
        {
            Id = user.Id,
            FirstName = user.FirstName,
            LastName = user.LastName,
            Email = user.Email,
            CreatedAt = user.CreatedAt
        };

        return Results.Json(userV2);
    }
}

// Register handlers
builder.Services.AddScoped<IMediaTypeHandler, UserV1Handler>();
builder.Services.AddScoped<IMediaTypeHandler, UserV2Handler>();

// Use in endpoint
app.MapGet("/users/{id}", async (
    HttpContext context,
    IEnumerable<IMediaTypeHandler> handlers) =>
{
    var accept = context.Request.Headers.Accept.ToString();

    var handler = handlers.FirstOrDefault(h => h.CanHandle(accept));

    if (handler is null)
    {
        return Results.BadRequest(new
        {
            Error = "Unsupported media type",
            Supported = new[]
            {
                "application/vnd.myapp.user.v1+json",
                "application/vnd.myapp.user.v2+json"
            }
        });
    }

    return await handler.HandleAsync(context);
});
```

## OpenAPI Documentation

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API v1",
        Version = "v1",
        Description = "Version 1 of the API (deprecated)"
    });

    options.SwaggerDoc("v2", new OpenApiInfo
    {
        Title = "My API v2",
        Version = "v2",
        Description = "Version 2 of the API (current)"
    });
});

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI(options =>
{
    options.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1");
    options.SwaggerEndpoint("/swagger/v2/swagger.json", "My API v2");
});

// Tag endpoints with version
app.MapGet("/users", GetUsersV1)
    .WithTags("Users")
    .WithGroupName("v1")
    .WithMetadata(new ProducesAttribute("application/vnd.myapp.v1+json"));

app.MapGet("/users", GetUsersV2)
    .WithTags("Users")
    .WithGroupName("v2")
    .WithMetadata(new ProducesAttribute("application/vnd.myapp.v2+json"));
```

## Testing Media Type Versioning

```csharp
public class MediaTypeVersioningTests
{
    [Fact]
    public async Task GetUsers_V1MediaType_ReturnsV1Format()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();
        client.DefaultRequestHeaders.Accept.Add(
            new MediaTypeWithQualityHeaderValue("application/vnd.myapp.v1+json"));

        // Act
        var response = await client.GetAsync("/users");

        // Assert
        response.EnsureSuccessStatusCode();

        var content = await response.Content.ReadAsStringAsync();
        var users = JsonSerializer.Deserialize<UserV1[]>(content);

        Assert.NotNull(users);
        Assert.All(users, u => Assert.IsType<int>(u.Id));
    }

    [Fact]
    public async Task GetUsers_V2MediaType_ReturnsV2Format()
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();
        client.DefaultRequestHeaders.Accept.Add(
            new MediaTypeWithQualityHeaderValue("application/vnd.myapp.v2+json"));

        // Act
        var response = await client.GetAsync("/users");

        // Assert
        response.EnsureSuccessStatusCode();

        var content = await response.Content.ReadAsStringAsync();
        var users = JsonSerializer.Deserialize<UserV2[]>(content);

        Assert.NotNull(users);
        Assert.All(users, u =>
        {
            Assert.IsType<string>(u.Id);
            Assert.NotEmpty(u.Email);
        });
    }
}
```

## Guidelines

**Media Type Format:**
- Use vendor prefix: `application/vnd.{company}.{resource}.{version}+{format}`
- Example: `application/vnd.myapp.user.v1+json`
- Support common formats: +json, +xml

**Versioning Strategy:**
- Default to latest version if no Accept header
- Support quality values (q=) for preference
- Add deprecation warnings for old versions
- Document all supported media types

**Backwards Compatibility:**
- Don't remove old versions immediately
- Provide migration guide
- Use Sunset header for removal date
- Support parallel versions during transition

**Client Communication:**
- Document media types in API docs
- Provide examples for each version
- Clear migration path
- Version-specific SDKs or clients

## Benefits

Clean URLs. No version in path.

Client-driven. Clients choose version via headers.

RESTful. Follows HTTP standards.

Flexible. Multiple versions simultaneously.

## Related

- [api-versioning-strategies.md](./api-versioning-strategies.md) - Other versioning approaches
- [content-negotiation.md](./content-negotiation.md) - Format negotiation
- [swagger-openapi.md](./swagger-openapi.md) - API documentation
- [api-backwards-compatibility.md](../05-aspnet-advanced/api-backwards-compatibility.md) - Compatibility strategies
