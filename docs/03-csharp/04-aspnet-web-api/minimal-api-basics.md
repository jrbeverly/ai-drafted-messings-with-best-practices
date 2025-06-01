# Minimal API Basics

Minimal APIs provide a simplified approach to building HTTP APIs with ASP.NET Core. No controllers, less ceremony.

## Principle

One endpoint per file. Static classes. Explicit route mapping. Fast startup.

## Basic Structure

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure middleware
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// Map endpoints
app.MapGet("/", () => "Hello World!");

app.Run();
```

## Endpoint Definition

Single file per endpoint:

```csharp
// Endpoints/Users/GetUserEndpoint.cs
public static class GetUserEndpoint
{
    public static void MapGetUser(this IEndpointRouteBuilder app)
    {
        app.MapGet("/api/users/{id}", async (
            string id,
            IUserService userService) =>
        {
            var user = await userService.GetUserByIdAsync(id);

            return user is not null
                ? Results.Ok(user)
                : Results.NotFound();
        })
        .WithName("GetUser")
        .WithTags("Users")
        .Produces<UserResponse>(200)
        .Produces(404);
    }
}

// Program.cs
app.MapGetUser();
```

## Route Parameters

```csharp
// Path parameters
app.MapGet("/users/{id}", (string id) => $"User {id}");

// Query parameters
app.MapGet("/users", (int page, int limit) =>
    $"Page {page}, Limit {limit}");

// Multiple parameters
app.MapGet("/users/{id}/loans/{loanId}",
    (string id, string loanId) =>
    $"User {id}, Loan {loanId}");
```

## Request Binding

```csharp
// From body
app.MapPost("/users", async (
    CreateUserRequest request,
    IUserService userService) =>
{
    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});

// From header
app.MapGet("/users/me", async (
    [FromHeader(Name = "Authorization")] string authorization,
    IUserService userService) =>
{
    // Extract user from token
    var userId = ExtractUserId(authorization);
    var user = await userService.GetUserByIdAsync(userId);
    return Results.Ok(user);
});

// From services (DI)
app.MapGet("/users", async (
    IUserService userService,
    ILogger<Program> logger) =>
{
    logger.LogInformation("Fetching all users");
    var users = await userService.GetUsersAsync();
    return Results.Ok(users);
});
```

## Response Types

```csharp
// OK (200)
app.MapGet("/users/{id}", async (string id, IUserService service) =>
{
    var user = await service.GetUserByIdAsync(id);
    return Results.Ok(user);
});

// Created (201)
app.MapPost("/users", async (CreateUserRequest request, IUserService service) =>
{
    var user = await service.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});

// NoContent (204)
app.MapDelete("/users/{id}", async (string id, IUserService service) =>
{
    await service.DeleteUserAsync(id);
    return Results.NoContent();
});

// NotFound (404)
app.MapGet("/users/{id}", async (string id, IUserService service) =>
{
    var user = await service.GetUserByIdAsync(id);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});

// BadRequest (400)
app.MapPost("/users", (CreateUserRequest request) =>
{
    if (string.IsNullOrEmpty(request.Email))
    {
        return Results.BadRequest(new { error = "Email is required" });
    }
    return Results.Ok();
});
```

## Validation

```csharp
// Manual validation
app.MapPost("/users", async (
    CreateUserRequest request,
    IUserService userService) =>
{
    if (string.IsNullOrEmpty(request.Email))
    {
        return Results.ValidationProblem(new Dictionary<string, string[]>
        {
            ["Email"] = new[] { "Email is required" }
        });
    }

    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});

// FluentValidation filter
app.MapPost("/users", async (
    CreateUserRequest request,
    IValidator<CreateUserRequest> validator,
    IUserService userService) =>
{
    var validationResult = await validator.ValidateAsync(request);

    if (!validationResult.IsValid)
    {
        return Results.ValidationProblem(
            validationResult.ToDictionary());
    }

    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});
```

## Route Groups

```csharp
// Group related endpoints
var users = app.MapGroup("/api/users")
    .WithTags("Users")
    .WithOpenApi();

users.MapGet("/", GetAllUsers);
users.MapGet("/{id}", GetUserById);
users.MapPost("/", CreateUser);
users.MapPut("/{id}", UpdateUser);
users.MapDelete("/{id}", DeleteUser);

// Nested groups
var userLoans = users.MapGroup("/{userId}/loans")
    .WithTags("Loans");

userLoans.MapGet("/", GetUserLoans);
userLoans.MapPost("/", CreateLoan);
```

## Filters

```csharp
// Endpoint filter
app.MapGet("/users/{id}", async (
    string id,
    IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);
    return Results.Ok(user);
})
.AddEndpointFilter(async (context, next) =>
{
    // Before endpoint
    var stopwatch = Stopwatch.StartNew();

    var result = await next(context);

    // After endpoint
    stopwatch.Stop();
    Console.WriteLine($"Execution time: {stopwatch.ElapsedMilliseconds}ms");

    return result;
});

// Global filter
builder.Services.AddSingleton<IEndpointFilter, LoggingFilter>();

public class LoggingFilter : IEndpointFilter
{
    private readonly ILogger<LoggingFilter> _logger;

    public LoggingFilter(ILogger<LoggingFilter> logger)
    {
        _logger = logger;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        _logger.LogInformation("Before endpoint execution");
        var result = await next(context);
        _logger.LogInformation("After endpoint execution");
        return result;
    }
}
```

## Error Handling

```csharp
// Try-catch in endpoint
app.MapGet("/users/{id}", async (
    string id,
    IUserService userService,
    ILogger<Program> logger) =>
{
    try
    {
        var user = await userService.GetUserByIdAsync(id);
        return Results.Ok(user);
    }
    catch (NotFoundException ex)
    {
        logger.LogWarning(ex, "User not found: {UserId}", id);
        return Results.NotFound();
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Error fetching user: {UserId}", id);
        return Results.Problem("An error occurred while fetching the user");
    }
});

// Global exception handler
app.UseExceptionHandler(exceptionHandlerApp =>
{
    exceptionHandlerApp.Run(async context =>
    {
        var exceptionHandlerFeature =
            context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionHandlerFeature?.Error;

        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "An error occurred",
            Detail = exception?.Message
        };

        context.Response.StatusCode =
            StatusCodes.Status500InternalServerError;
        await context.Response.WriteAsJsonAsync(problemDetails);
    });
});
```

## Guidelines

**Structure:**
- One endpoint per file
- Static classes for endpoint definitions
- Extension methods for registration
- Group related endpoints

**Parameters:**
- Use parameter binding (path, query, body, header)
- Inject services via DI
- Validate input early

**Responses:**
- Use Results.* for consistent responses
- Include status codes
- Return appropriate content types

**Organization:**
- Endpoints/{Resource}/{ActionEndpoint.cs}
- Map in Program.cs or endpoint modules
- Keep endpoints thin, delegate to services

## Benefits

Simple. Less boilerplate than controllers.

Fast. Minimal overhead, quick startup.

Explicit. Clear route definitions.

Testable. Easy to unit test.

## Related

- [endpoint-filters.md](./endpoint-filters.md) - Request/response filters
- [route-groups.md](./route-groups.md) - Organizing endpoints
- [request-validation.md](./request-validation.md) - Input validation
- [dependency-injection.md](../03-dotnet/dependency-injection.md) - Service injection
