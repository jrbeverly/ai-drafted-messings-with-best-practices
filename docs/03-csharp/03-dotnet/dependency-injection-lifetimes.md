# Dependency Injection Lifetimes

Choose between Singleton, Scoped, and Transient based on state, resource usage, and thread safety.

## Why It Matters

- Wrong lifetime causes captive dependencies, memory leaks, or thread-safety bugs
- Correct lifetime enables the DI container to manage creation and disposal automatically
- Understanding lifetimes prevents the most common .NET DI mistakes

## Lifetime Comparison

| Aspect | Singleton | Scoped | Transient |
|---|---|---|---|
| Instances | 1 per app | 1 per request/scope | 1 per injection |
| Thread-safe required | Yes | No | No |
| Use for | Caches, config, stateless shared services | Repositories, business logic, request context | Lightweight stateless utilities |
| Memory | Higher (lives forever) | Medium | Lower |

## Registration

```csharp
builder.Services.AddSingleton<ICacheService, CacheService>();
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddTransient<IEmailSender, EmailSender>();
```

## Default to Scoped

Most application services (repositories, business logic, services with dependencies) should be **Scoped**. Use Singleton only for stateless or thread-safe shared resources. Use Transient only for lightweight, stateless utilities.

## Captive Dependency (The Critical Bug)

A longer-lived service captures a shorter-lived dependency, preventing disposal:

```csharp
// BUG: Singleton captures Scoped -- userService is never disposed
public class CacheService(IUserService userService) { }  // Singleton
```

**Fix:** Use `IServiceScopeFactory` to create a scope on demand:

```csharp
public class CacheService(IServiceScopeFactory scopeFactory)
{
    public async Task DoWork()
    {
        using var scope = scopeFactory.CreateScope();
        var userService = scope.ServiceProvider.GetRequiredService<IUserService>();
        await userService.DoWorkAsync();
    }
}
```

## Safe Dependency Direction

- Singleton can depend on: Singleton only
- Scoped can depend on: Singleton, Scoped
- Transient can depend on: Singleton, Scoped, Transient

**Rule:** Never inject a shorter lifetime into a longer one.

## Pitfalls to Avoid

- Singleton with mutable non-thread-safe state (use `ConcurrentDictionary`, not `Dictionary`)
- Transient for expensive-to-create services (pay creation cost every injection)
- Injecting `IServiceProvider` directly (service locator anti-pattern)
- Registering scoped services and forgetting to create scopes in background services

## Related

- [dependency-injection.md](./dependency-injection.md) -- Basic DI patterns
- [background-services.md](./background-services.md) -- Scopes in background services
