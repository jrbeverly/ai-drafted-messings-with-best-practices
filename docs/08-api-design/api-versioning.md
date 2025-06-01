# API Versioning

Evolve APIs without breaking clients. Version strategies, migration patterns, deprecation policies.

## Principle

Version early. Support multiple versions. Deprecate gracefully. Communicate changes.

## Versioning Strategies

URL path versioning (recommended):

```csharp
// Version in URL path
app.MapGet("/v1/users/{id}", GetUserV1);
app.MapGet("/v2/users/{id}", GetUserV2);

// Clear and explicit
// Easy to route
// Visible in documentation
```

Header versioning:

```csharp
app.MapGet("/users/{id}", async (
    string id,
    HttpContext context,
    IUserService userService) =>
{
    var version = context.Request.Headers["API-Version"].FirstOrDefault() ?? "1";

    return version switch
    {
        "1" => await GetUserV1(id, userService),
        "2" => await GetUserV2(id, userService),
        _ => Results.BadRequest(new { error = "Unsupported API version" })
    };
});

// Request
// GET /users/123
// API-Version: 2
```

Accept header versioning:

```csharp
app.MapGet("/users/{id}", async (
    string id,
    HttpContext context,
    IUserService userService) =>
{
    var accept = context.Request.Headers["Accept"].ToString();

    if (accept.Contains("application/vnd.myapi.v2+json"))
    {
        return await GetUserV2(id, userService);
    }

    return await GetUserV1(id, userService);
});

// Request
// GET /users/123
// Accept: application/vnd.myapi.v2+json
```

## URL Path Versioning

Organize by version:

```
src/LibraryService/
└── LibraryService.Api/
    ├── Routes/
    │   ├── v1/
    │   │   ├── Users/
    │   │   │   ├── CreateUserRoute.cs
    │   │   │   ├── GetUserRoute.cs
    │   │   │   └── UpdateUserRoute.cs
    │   │   └── Loans/
    │   │       └── ...
    │   └── v2/
    │       ├── Users/
    │       │   ├── CreateUserRoute.cs
    │       │   ├── GetUserRoute.cs
    │       │   └── UpdateUserRoute.cs
    │       └── Loans/
    │           └── ...
    └── Program.cs
```

Register routes:

```csharp
// Program.cs
var app = builder.Build();

// API v1
var v1 = app.MapGroup("/api/v1").WithTags("v1");
v1.MapUsersV1();
v1.MapLoansV1();

// API v2
var v2 = app.MapGroup("/api/v2").WithTags("v2");
v2.MapUsersV2();
v2.MapLoansV2();

app.Run();
```

Route extensions:

```csharp
// Routes/v1/Users/UserRoutesV1.cs
namespace LibraryService.Routes.v1.Users;

public static class UserRoutesV1
{
    public static void MapUsersV1(this IEndpointRouteBuilder app)
    {
        app.MapGet("/users", GetUsersV1.HandleAsync)
            .WithName("GetUsers_V1")
            .WithDescription("Get all users (v1)");

        app.MapGet("/users/{id}", GetUserV1.HandleAsync)
            .WithName("GetUser_V1")
            .WithDescription("Get user by ID (v1)");

        app.MapPost("/users", CreateUserV1.HandleAsync)
            .WithName("CreateUser_V1")
            .WithDescription("Create new user (v1)");
    }
}

// Routes/v2/Users/UserRoutesV2.cs
namespace LibraryService.Routes.v2.Users;

public static class UserRoutesV2
{
    public static void MapUsersV2(this IEndpointRouteBuilder app)
    {
        app.MapGet("/users", GetUsersV2.HandleAsync)
            .WithName("GetUsers_V2")
            .WithDescription("Get all users (v2)");

        app.MapGet("/users/{id}", GetUserV2.HandleAsync)
            .WithName("GetUser_V2")
            .WithDescription("Get user by ID (v2)");

        app.MapPost("/users", CreateUserV2.HandleAsync)
            .WithName("CreateUser_V2")
            .WithDescription("Create new user (v2)");
    }
}
```

## Version-Specific DTOs

Separate DTOs per version:

```csharp
// DTOs/v1/UserDto.cs
namespace LibraryService.DTOs.v1;

public record UserResponse
{
    public required string Id { get; init; }
    public required string Email { get; init; }
    public required string Name { get; init; }
    public required DateTime CreatedAt { get; init; }
}

// DTOs/v2/UserDto.cs
namespace LibraryService.DTOs.v2;

public record UserResponse
{
    public required string Id { get; init; }
    public required string Email { get; init; }

    // Split name into first and last
    public required string FirstName { get; init; }
    public required string LastName { get; init; }

    // New fields in v2
    public string? PhoneNumber { get; init; }
    public required DateTime CreatedAt { get; init; }
    public DateTime? UpdatedAt { get; init; }
}
```

Mapping between versions:

```csharp
// Mappers/UserMapper.cs
public static class UserMapper
{
    public static DTOs.v1.UserResponse ToV1Response(User user)
    {
        return new DTOs.v1.UserResponse
        {
            Id = user.Id,
            Email = user.Email,
            Name = user.Name,
            CreatedAt = user.CreatedAt
        };
    }

    public static DTOs.v2.UserResponse ToV2Response(User user)
    {
        var nameParts = user.Name.Split(' ', 2);

        return new DTOs.v2.UserResponse
        {
            Id = user.Id,
            Email = user.Email,
            FirstName = nameParts.Length > 0 ? nameParts[0] : user.Name,
            LastName = nameParts.Length > 1 ? nameParts[1] : string.Empty,
            PhoneNumber = user.PhoneNumber,
            CreatedAt = user.CreatedAt,
            UpdatedAt = user.UpdatedAt
        };
    }
}
```

## Shared Business Logic

Reuse domain logic across versions:

```csharp
// Application/Users/GetUser/GetUserHandler.cs
public class GetUserHandler
{
    private readonly IUserRepository _repository;

    public GetUserHandler(IUserRepository repository)
    {
        _repository = repository;
    }

    // Shared business logic
    public async Task<User?> HandleAsync(string userId)
    {
        return await _repository.GetByIdAsync(userId);
    }
}

// Routes/v1/Users/GetUserV1.cs
public static class GetUserV1
{
    public static async Task<IResult> HandleAsync(
        string id,
        GetUserHandler handler)
    {
        var user = await handler.HandleAsync(id);

        if (user == null)
            return Results.NotFound();

        // V1-specific response
        return Results.Ok(UserMapper.ToV1Response(user));
    }
}

// Routes/v2/Users/GetUserV2.cs
public static class GetUserV2
{
    public static async Task<IResult> HandleAsync(
        string id,
        GetUserHandler handler)
    {
        var user = await handler.HandleAsync(id);

        if (user == null)
            return Results.NotFound();

        // V2-specific response
        return Results.Ok(UserMapper.ToV2Response(user));
    }
}
```

## Deprecation Strategy

Mark deprecated versions:

```csharp
// Program.cs
var v1 = app.MapGroup("/api/v1")
    .WithTags("v1 (Deprecated)")
    .WithMetadata(new DeprecatedAttribute
    {
        DeprecatedOn = new DateTime(2024, 1, 1),
        RemovalDate = new DateTime(2024, 6, 1),
        Message = "Use /api/v2 instead"
    });
```

Deprecation middleware:

```csharp
public class DeprecationMiddleware
{
    private readonly RequestDelegate _next;

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Path.StartsWithSegments("/api/v1"))
        {
            context.Response.Headers["Deprecated"] = "true";
            context.Response.Headers["Sunset"] = "Mon, 01 Jun 2024 00:00:00 GMT";
            context.Response.Headers["Link"] = "</api/v2>; rel=\"successor-version\"";
        }

        await _next(context);
    }
}

// Register
app.UseMiddleware<DeprecationMiddleware>();
```

## Breaking Changes

Identify breaking changes:

```csharp
// Breaking changes (require new version)
// - Removing fields
// - Renaming fields
// - Changing field types
// - Removing endpoints
// - Changing validation rules (stricter)

// Non-breaking changes (can stay in same version)
// - Adding new fields (optional)
// - Adding new endpoints
// - Relaxing validation rules
// - Adding new optional query parameters
```

Example migration:

```csharp
// V1 - Original
public record UserResponseV1
{
    public required string Id { get; init; }
    public required string FullName { get; init; }
}

// V2 - Breaking change: split name
public record UserResponseV2
{
    public required string Id { get; init; }
    public required string FirstName { get; init; }  // NEW
    public required string LastName { get; init; }   // NEW
    // FullName removed - BREAKING
}

// V3 - Non-breaking: add optional field
public record UserResponseV3
{
    public required string Id { get; init; }
    public required string FirstName { get; init; }
    public required string LastName { get; init; }
    public string? MiddleName { get; init; }  // NEW, optional - NOT BREAKING
}
```

## Default Version

Specify default version:

```csharp
// Redirect root to latest version
app.MapGet("/api/users", () =>
    Results.Redirect("/api/v2/users", permanent: false));

// Or handle with header
app.MapGet("/api/users", async (
    HttpContext context,
    GetUsersHandler handler) =>
{
    var version = context.Request.Headers["API-Version"].FirstOrDefault() ?? "2";

    return version switch
    {
        "1" => await GetUsersV1(handler),
        "2" => await GetUsersV2(handler),
        _ => Results.BadRequest(new { error = "Unsupported version" })
    };
});
```

## Version Discovery

Document available versions:

```csharp
// GET /api/versions
app.MapGet("/api/versions", () => Results.Ok(new
{
    versions = new[]
    {
        new
        {
            version = "v1",
            status = "deprecated",
            deprecatedOn = "2024-01-01",
            sunsetDate = "2024-06-01",
            documentation = "https://api.example.com/docs/v1"
        },
        new
        {
            version = "v2",
            status = "current",
            documentation = "https://api.example.com/docs/v2"
        }
    },
    current = "v2"
}));
```

## Testing Multiple Versions

Test each version independently:

```csharp
// Tests/v1/UserEndpointsV1Tests.cs
public class UserEndpointsV1Tests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public UserEndpointsV1Tests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetUser_ReturnsV1Format()
    {
        var response = await _client.GetAsync("/api/v1/users/user_123");

        response.EnsureSuccessStatusCode();

        var user = await response.Content.ReadFromJsonAsync<DTOs.v1.UserResponse>();

        Assert.NotNull(user);
        Assert.NotNull(user.Name);  // V1 has single name field
    }
}

// Tests/v2/UserEndpointsV2Tests.cs
public class UserEndpointsV2Tests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public UserEndpointsV2Tests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetUser_ReturnsV2Format()
    {
        var response = await _client.GetAsync("/api/v2/users/user_123");

        response.EnsureSuccessStatusCode();

        var user = await response.Content.ReadFromJsonAsync<DTOs.v2.UserResponse>();

        Assert.NotNull(user);
        Assert.NotNull(user.FirstName);  // V2 has split name
        Assert.NotNull(user.LastName);
    }
}
```

## Documentation Per Version

Separate OpenAPI docs:

```csharp
// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    // V1 documentation
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Library API v1",
        Version = "v1",
        Description = "Version 1 of the Library API (Deprecated)"
    });

    // V2 documentation
    options.SwaggerDoc("v2", new OpenApiInfo
    {
        Title = "Library API v2",
        Version = "v2",
        Description = "Version 2 of the Library API (Current)"
    });

    // Filter endpoints by version
    options.DocInclusionPredicate((docName, apiDesc) =>
    {
        return apiDesc.RelativePath?.Contains($"/{docName}/") ?? false;
    });
});

app.UseSwagger();
app.UseSwaggerUI(options =>
{
    options.SwaggerEndpoint("/swagger/v1/swagger.json", "Library API v1");
    options.SwaggerEndpoint("/swagger/v2/swagger.json", "Library API v2");
});
```

## Migration Guide

Document migration path:

```markdown
# Migration Guide: v1 to v2

## Breaking Changes

### User Name Split
V1 returned a single `name` field. V2 splits this into `firstName` and `lastName`.

**V1 Response:**
```json
{
  "id": "user_123",
  "name": "John Doe",
  "email": "john@example.com"
}
```

**V2 Response:**
```json
{
  "id": "user_123",
  "firstName": "John",
  "lastName": "Doe",
  "email": "john@example.com"
}
```

**Migration:**
- Split `name` field on first space
- Handle names with no spaces (use as `firstName`, empty `lastName`)

### New Required Fields
- `phoneNumber` is now optional but recommended

## Timeline
- V1 deprecated: January 1, 2024
- V1 sunset: June 1, 2024
- All clients must migrate before June 1, 2024
```

## Guidelines

**Versioning Strategy:**
- URL path versioning for clarity
- Major versions only (v1, v2, v3)
- Never break existing versions
- Support at least 2 versions simultaneously

**Breaking Changes:**
- Require new major version
- Document all breaking changes
- Provide migration guide
- Give 6+ months notice before sunset

**Non-Breaking Changes:**
- Add to existing version
- Make new fields optional
- Relax validation, don't tighten
- Add new endpoints freely

**Deprecation:**
- Announce deprecation early
- Set sunset date (6-12 months)
- Use deprecation headers
- Document successor version

**Testing:**
- Test all supported versions
- Separate test suites per version
- Test migration paths
- Monitor version usage

## Benefits

Backward compatible. Old clients keep working.

Gradual migration. Clients upgrade on their schedule.

Clear expectations. Versions communicate stability.

Safe evolution. New features without breaking changes.

## Related

- [rest-api-principles.md](./rest-api-principles.md) - API design patterns
- [api-error-handling.md](./api-error-handling.md) - Error handling
- [openapi-documentation.md](./openapi-documentation.md) - API documentation
