# Top-Level Statements

C# 9 top-level statements remove ceremony from Program.cs. No explicit Main method or class needed.

## Principle

Entry point code written directly without class or method wrapper. Reduces boilerplate for simple programs.

## Basic Syntax

Minimal API program:

```csharp
// Program.cs - No class, no Main method
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

Compiler generates Main method automatically.

## Traditional vs Top-Level

Traditional approach:

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);

        builder.Services.AddEndpointsApiExplorer();
        builder.Services.AddSwaggerGen();

        var app = builder.Build();

        app.MapGet("/", () => "Hello World!");

        app.Run();
    }
}
```

Top-level statements:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

Same functionality, less ceremony.

## Minimal API Example

Complete API with top-level statements:

```csharp
using LibraryService.Infrastructure;
using LibraryService.Routes.Users.v1;

var builder = WebApplication.CreateBuilder(args);

// Configure services
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddInfrastructure(builder.Configuration);

var app = builder.Build();

// Configure middleware
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// Map routes
var v1 = app.MapGroup("/api/v1");
var users = v1.MapGroup("/users");

UserGetRoute.Registration.Map(users);
UserCreateRoute.Registration.Map(users);

app.Run();
```

## Service Registration

Extract service registration to extension methods:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddInfrastructure(builder.Configuration);
builder.Services.AddDomain();

var app = builder.Build();

// ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<ILoanRepository, LoanRepository>();
        services.AddSingleton<ICacheService, CacheService>();

        return services;
    }

    public static IServiceCollection AddDomain(
        this IServiceCollection services)
    {
        services.AddScoped<IUserService, UserService>();
        services.AddScoped<ILoanService, LoanService>();

        return services;
    }
}
```

## Route Registration

Extract route registration:

```csharp
// Program.cs
var app = builder.Build();

app.MapRoutes();

app.Run();

// WebApplicationExtensions.cs
public static class WebApplicationExtensions
{
    public static WebApplication MapRoutes(this WebApplication app)
    {
        var v1 = app.MapGroup("/api/v1");

        MapUserRoutes(v1);
        MapLoanRoutes(v1);

        return app;
    }

    private static void MapUserRoutes(RouteGroupBuilder group)
    {
        var users = group.MapGroup("/users");

        UserGetRoute.Registration.Map(users);
        UserCreateRoute.Registration.Map(users);
        UserUpdateRoute.Registration.Map(users);
    }

    private static void MapLoanRoutes(RouteGroupBuilder group)
    {
        var loans = group.MapGroup("/loans/{libraryId}");

        LoanCreateRoute.Registration.Map(loans);
        LoanGetRoute.Registration.Map(loans);
    }
}
```

## Full Example

Complete Program.cs with top-level statements:

```csharp
using LibraryService.Infrastructure;
using LibraryService.Middleware;

var builder = WebApplication.CreateBuilder(args);

// Services
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddAuthentication();
builder.Services.AddAuthorization();

// Custom services
builder.Services.AddInfrastructure(builder.Configuration);
builder.Services.AddDomain();

var app = builder.Build();

// Middleware
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

// Exception handling
app.UseMiddleware<DomainExceptionMiddleware>();

// Routes
app.MapRoutes();

app.Run();
```

Clean, sequential, easy to understand.

## Accessing Program Class

For testing, expose Program class:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// ... configuration ...

var app = builder.Build();

// ... middleware and routes ...

app.Run();

// Make Program accessible for testing
public partial class Program { }
```

Integration tests:

```csharp
public class ApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public ApiTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    [Fact]
    public async Task GetUser_ReturnsSuccess()
    {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/api/v1/users/123");

        response.EnsureSuccessStatusCode();
    }
}
```

## Local Functions

Define helper functions:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddInfrastructure(builder.Configuration);

var app = builder.Build();

app.MapGet("/health", GetHealthStatus);
app.MapGet("/version", () => GetVersion());

app.Run();

// Local functions
static IResult GetHealthStatus()
{
    return Results.Ok(new { Status = "Healthy" });
}

static string GetVersion()
{
    return "1.0.0";
}
```

## Guidelines

**Use Top-Level Statements:**
- New projects (default for modern .NET)
- Minimal APIs
- Simple console applications
- Reduces boilerplate

**Extract to Methods:**
- Complex service registration
- Route configuration
- Middleware setup
- Keep Program.cs focused

**Testing:**
- Use partial class declaration
- Expose Program for WebApplicationFactory
- Test via integration tests

**Structure:**
1. Service registration
2. App building
3. Middleware configuration
4. Route mapping
5. App.Run()

## Benefits

Less ceremony. No explicit class or Main method.

Easier to read. Sequential flow is clear.

Modern idiom. Default for new .NET projects.

## Related

- [file-scoped-namespaces.md](./file-scoped-namespaces.md) - Reduce namespace boilerplate
- [global-usings.md](./global-usings.md) - Reduce using statements
- [minimal-api-routing.md](../05-aspnet-web-apis/minimal-api-routing.md) - Route registration
