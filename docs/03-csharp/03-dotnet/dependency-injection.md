# Dependency Injection

Register services in the built-in DI container and inject them via constructors.

## Why It Matters

- Loose coupling: services depend on interfaces, not concrete implementations
- Testability: swap real implementations for mocks in tests
- Lifetime management: container handles creation and disposal automatically

## Registration

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IUserRepository, DynamoDbUserRepository>();
builder.Services.AddSingleton<ICacheService, CacheService>();
builder.Services.AddTransient<IEmailSender, EmailSender>();
```

## Constructor Injection

```csharp
public class UserService(IUserRepository repository, ILogger<UserService> logger) : IUserService
{
    public async Task<User> GetUserAsync(Guid id)
    {
        logger.LogInformation("Getting user {UserId}", id);
        return await repository.GetByIdAsync(id);
    }
}
```

## Minimal API Endpoint Injection

```csharp
app.MapGet("/users/{id}", async (Guid id, IUserService userService) =>
{
    var user = await userService.GetUserAsync(id);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});
```

## Multiple Implementations

```csharp
builder.Services.AddScoped<INotificationSender, EmailSender>();
builder.Services.AddScoped<INotificationSender, SmsSender>();

// Inject IEnumerable<INotificationSender> to get all implementations
```

## Factory Registration

```csharp
builder.Services.AddScoped<IUserService>(provider =>
{
    var config = provider.GetRequiredService<IConfiguration>();
    return config.GetValue<bool>("Features:EnableCaching")
        ? new CachedUserService(provider.GetRequiredService<IUserRepository>())
        : new UserService(provider.GetRequiredService<IUserRepository>());
});
```

## Extension Methods for Clean Registration

```csharp
public static IServiceCollection AddUserServices(this IServiceCollection services)
{
    services.AddScoped<IUserService, UserService>();
    services.AddScoped<IUserRepository, DynamoDbUserRepository>();
    return services;
}

// Program.cs
builder.Services.AddUserServices();
```

## Key Recommendations

- **Register by interface**, not concrete type
- **Default to Scoped** for most services
- **Keep constructors simple** -- assignment only, no logic
- **Use extension methods** to group related registrations

## Pitfalls to Avoid

- Injecting `IServiceProvider` directly (service locator anti-pattern)
- Circular dependencies (A depends on B depends on A) -- refactor to break the cycle
- Injecting scoped services into singletons (captive dependency)
- Complex logic in constructors -- move to initialization methods

## Related

- [dependency-injection-lifetimes.md](./dependency-injection-lifetimes.md) -- Detailed lifetime guidance
- [background-services.md](./background-services.md) -- DI in background services
