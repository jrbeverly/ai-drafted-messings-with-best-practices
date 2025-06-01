# Primary Constructors

C# 12 feature that declares constructor parameters directly in the class/struct declaration, reducing DI boilerplate.

## Why It Matters

- Eliminates field declarations + constructor assignment boilerplate
- Dependencies are visible in the class declaration line
- Natural fit for dependency injection services

## Key Recommendations

**Use for DI-based services:**
```csharp
public class UserService(IUserRepository repository, ILogger<UserService> logger) : IUserService
{
    public async Task<User> GetAsync(Guid id)
    {
        logger.LogInformation("Getting user {Id}", id);
        return await repository.GetByIdAsync(id);
    }
}
```

**Capture to `readonly` field when needed** (many-method access or readonly semantics):
```csharp
public class Service(ILogger<Service> logger)
{
    private readonly ILogger<Service> _logger = logger;
}
```

**Pass to base class:**
```csharp
public class UserService(IUserRepository repo, ILogger<UserService> logger)
    : ServiceBase(logger), IUserService { }
```

**Validate via property initialization:**
```csharp
public class Config(string apiKey)
{
    public string ApiKey { get; } = !string.IsNullOrEmpty(apiKey)
        ? apiKey : throw new ArgumentException("Required", nameof(apiKey));
}
```

**Additional constructors must call the primary:**
```csharp
public UserService(IUserRepository repo)
    : this(repo, NullLogger<UserService>.Instance) { }
```

## When to Use vs Traditional Constructors

| Use Primary | Use Traditional |
|-------------|-----------------|
| Simple DI parameter assignment | Complex initialization logic |
| Parameters used directly in methods | Extensive parameter validation |
| Reducing boilerplate | Multiple constructor overloads with logic |

## Pitfalls to Avoid

- Primary constructor parameters are mutable captures (not `readonly`) unless explicitly assigned to a `readonly` field
- Mixing direct parameter usage with field captures without clear reason
