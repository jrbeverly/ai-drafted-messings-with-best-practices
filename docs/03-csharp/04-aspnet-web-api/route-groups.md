# Route Groups

Route groups organize related endpoints under a common prefix. Share configuration across multiple routes.

## Principle

Group by resource. Apply common filters, metadata, and configuration to all routes in the group.

## Basic Group

```csharp
var users = app.MapGroup("/api/users");

users.MapGet("/", GetAllUsers);
users.MapGet("/{id}", GetUserById);
users.MapPost("/", CreateUser);
users.MapPut("/{id}", UpdateUser);
users.MapDelete("/{id}", DeleteUser);

// Resulting routes:
// GET    /api/users
// GET    /api/users/{id}
// POST   /api/users
// PUT    /api/users/{id}
// DELETE /api/users/{id}
```

## Group with Metadata

```csharp
var users = app.MapGroup("/api/users")
    .WithTags("Users")
    .WithOpenApi()
    .WithDescription("User management endpoints");

users.MapGet("/", GetAllUsers)
    .WithSummary("Get all users")
    .WithDescription("Returns a paginated list of users");

users.MapGet("/{id}", GetUserById)
    .WithSummary("Get user by ID")
    .Produces<UserResponse>(200)
    .Produces(404);
```

## Group with Filters

```csharp
var users = app.MapGroup("/api/users")
    .AddEndpointFilter<LoggingFilter>()
    .AddEndpointFilter<RateLimitFilter>();

// All endpoints in group have logging and rate limiting
users.MapGet("/", GetAllUsers);
users.MapPost("/", CreateUser);
```

## Nested Groups

```csharp
var api = app.MapGroup("/api");

// Users group under /api
var users = api.MapGroup("/users")
    .WithTags("Users");

users.MapGet("/", GetAllUsers);           // GET /api/users
users.MapGet("/{id}", GetUserById);       // GET /api/users/{id}

// Loans under specific user
var userLoans = users.MapGroup("/{userId}/loans")
    .WithTags("Loans");

userLoans.MapGet("/", GetUserLoans);      // GET /api/users/{userId}/loans
userLoans.MapPost("/", CreateLoan);       // POST /api/users/{userId}/loans
userLoans.MapGet("/{loanId}", GetLoan);   // GET /api/users/{userId}/loans/{loanId}
```

## Authorization Groups

```csharp
// Public endpoints (no auth required)
var publicApi = app.MapGroup("/api/public");

publicApi.MapGet("/books", GetPublicBooks);
publicApi.MapGet("/books/{id}", GetPublicBook);

// Authenticated endpoints
var authenticatedApi = app.MapGroup("/api")
    .RequireAuthorization();

authenticatedApi.MapGet("/users/me", GetCurrentUser);
authenticatedApi.MapPost("/loans", CreateLoan);

// Admin-only endpoints
var adminApi = app.MapGroup("/api/admin")
    .RequireAuthorization("AdminPolicy");

adminApi.MapGet("/users", GetAllUsers);
adminApi.MapDelete("/users/{id}", DeleteUser);
```

## Versioned Groups

```csharp
// Version 1
var v1 = app.MapGroup("/api/v1")
    .WithTags("V1");

v1.MapGet("/users", GetUsersV1);
v1.MapGet("/users/{id}", GetUserByIdV1);

// Version 2 with breaking changes
var v2 = app.MapGroup("/api/v2")
    .WithTags("V2");

v2.MapGet("/users", GetUsersV2);
v2.MapGet("/users/{id}", GetUserByIdV2);
```

## CORS Groups

```csharp
// Group with CORS policy
var publicApi = app.MapGroup("/api/public")
    .RequireCors("AllowPublicApi");

publicApi.MapGet("/books", GetBooks);
publicApi.MapGet("/authors", GetAuthors);

// CORS configuration
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowPublicApi", policy =>
    {
        policy.WithOrigins("https://example.com")
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});
```

## Rate Limit Groups

```csharp
// High rate limit for authenticated users
var authenticatedApi = app.MapGroup("/api")
    .RequireAuthorization()
    .RequireRateLimiting("authenticated");

// Low rate limit for public endpoints
var publicApi = app.MapGroup("/api/public")
    .RequireRateLimiting("public");

// Rate limiter configuration
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("public", options =>
    {
        options.PermitLimit = 10;
        options.Window = TimeSpan.FromMinutes(1);
    });

    options.AddFixedWindowLimiter("authenticated", options =>
    {
        options.PermitLimit = 100;
        options.Window = TimeSpan.FromMinutes(1);
    });
});
```

## Caching Groups

```csharp
var publicApi = app.MapGroup("/api/public")
    .CacheOutput(policy => policy.Expire(TimeSpan.FromMinutes(5)));

// All public endpoints cached for 5 minutes
publicApi.MapGet("/books", GetBooks);
publicApi.MapGet("/authors", GetAuthors);
```

## Module-Based Organization

```csharp
// Endpoints/Users/UserEndpoints.cs
public static class UserEndpoints
{
    public static RouteGroupBuilder MapUserEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/users")
            .WithTags("Users")
            .RequireAuthorization();

        group.MapGet("/", GetAllUsers);
        group.MapGet("/{id}", GetUserById);
        group.MapPost("/", CreateUser);
        group.MapPut("/{id}", UpdateUser);
        group.MapDelete("/{id}", DeleteUser);

        return group;
    }

    private static async Task<IResult> GetAllUsers(
        IUserService userService) =>
        Results.Ok(await userService.GetUsersAsync());

    private static async Task<IResult> GetUserById(
        string id,
        IUserService userService)
    {
        var user = await userService.GetUserByIdAsync(id);
        return user is not null ? Results.Ok(user) : Results.NotFound();
    }

    // Other handlers...
}

// Program.cs
app.MapUserEndpoints();
```

## Feature-Based Organization

```csharp
// Features/Users/UserModule.cs
public static class UserModule
{
    public static IServiceCollection AddUserServices(
        this IServiceCollection services)
    {
        services.AddScoped<IUserService, UserService>();
        services.AddScoped<IUserRepository, UserRepository>();
        return services;
    }

    public static IEndpointRouteBuilder MapUserEndpoints(
        this IEndpointRouteBuilder app)
    {
        var users = app.MapGroup("/api/users")
            .WithTags("Users")
            .RequireAuthorization()
            .AddEndpointFilter<ValidationFilter>();

        users.MapGet("/", GetAllUsers);
        users.MapGet("/{id}", GetUserById);
        users.MapPost("/", CreateUser);
        users.MapPut("/{id}", UpdateUser);
        users.MapDelete("/{id}", DeleteUser);

        return app;
    }

    // Handler methods...
}

// Program.cs
builder.Services.AddUserServices();
app.MapUserEndpoints();
```

## Conditional Groups

```csharp
if (app.Environment.IsDevelopment())
{
    var dev = app.MapGroup("/dev")
        .WithTags("Development");

    dev.MapGet("/seed", SeedDatabase);
    dev.MapGet("/reset", ResetDatabase);
}

if (app.Environment.IsProduction())
{
    var health = app.MapGroup("/health")
        .AllowAnonymous()
        .CacheOutput(policy => policy.Expire(TimeSpan.FromSeconds(10)));

    health.MapGet("/", () => Results.Ok(new { status = "healthy" }));
}
```

## Shared Prefix Patterns

```csharp
// Organization: /api/{organization}/...
var orgApi = app.MapGroup("/api/{organizationId}")
    .AddEndpointFilter(async (context, next) =>
    {
        // Validate organization exists and user has access
        var orgId = context.HttpContext.GetRouteValue("organizationId");
        // Validation logic...
        return await next(context);
    });

var users = orgApi.MapGroup("/users");
users.MapGet("/", GetOrgUsers);          // GET /api/{organizationId}/users
users.MapGet("/{id}", GetOrgUser);       // GET /api/{organizationId}/users/{id}

var projects = orgApi.MapGroup("/projects");
projects.MapGet("/", GetOrgProjects);    // GET /api/{organizationId}/projects
projects.MapPost("/", CreateOrgProject); // POST /api/{organizationId}/projects
```

## Guidelines

**Organization:**
- Group by resource or feature
- Use nested groups for sub-resources
- Keep groups focused (single responsibility)

**Configuration:**
- Apply common filters to groups
- Set metadata at group level
- Use groups for auth, CORS, rate limiting

**Naming:**
- Use plural nouns for collections (`/users`, not `/user`)
- RESTful conventions (`/users/{id}`, not `/getUser/{id}`)
- Consistent casing (prefer kebab-case for URLs)

**Versioning:**
- Group by version (`/api/v1`, `/api/v2`)
- Maintain old versions during migration
- Document breaking changes

## Benefits

Organized. Related endpoints grouped together.

DRY. Share configuration across endpoints.

Maintainable. Easy to add/remove endpoints.

Discoverable. Clear API structure.

## Related

- [minimal-api-basics.md](./minimal-api-basics.md) - Endpoint definition
- [endpoint-filters.md](./endpoint-filters.md) - Filters applied to groups
- [api-versioning.md](../../08-api-design/api-versioning.md) - Version strategies
- [rest-api-principles.md](../../08-api-design/rest-api-principles.md) - RESTful design
