# Configuration Options Pattern

Strongly-typed configuration with `IOptions<T>` -- bind config sections to POCOs with validation.

## Why It Matters

- Compile-time type safety and IDE autocomplete for all configuration
- Fail-fast at startup when configuration is invalid
- Easy to test by injecting options directly

## Basic Pattern

```csharp
public class DatabaseOptions
{
    public const string SectionName = "Database";

    [Required]
    public string TableName { get; set; } = string.Empty;
    [Range(1, 300)]
    public int TimeoutSeconds { get; set; } = 30;
}

// Registration with validation
builder.Services.AddOptions<DatabaseOptions>()
    .Bind(builder.Configuration.GetSection(DatabaseOptions.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

## Three Options Interfaces

| Interface | Lifetime | Reloads | Use When |
|---|---|---|---|
| `IOptions<T>` | Singleton | No | Config never changes at runtime |
| `IOptionsSnapshot<T>` | Scoped | Per request | Config changes occasionally |
| `IOptionsMonitor<T>` | Singleton | Real-time | Config changes frequently, need `OnChange` callback |

## Injection

```csharp
public class UserRepository(IOptions<DatabaseOptions> options)
{
    private readonly DatabaseOptions _opts = options.Value;
}
```

## Environment Variable Overrides

```bash
# Use double underscore for hierarchy
export Database__TableName=Users
export Database__TimeoutSeconds=60
```

Environment variables override `appsettings.json` values.

## Named Options (Multiple Configs of Same Type)

```csharp
builder.Services.Configure<DatabaseOptions>("Primary", config.GetSection("Database:Primary"));
builder.Services.Configure<DatabaseOptions>("Secondary", config.GetSection("Database:Secondary"));

// Resolve with IOptionsSnapshot<T>
var primary = options.Get("Primary");
```

## Pitfalls to Avoid

- Accessing `Configuration["Key"]` strings instead of typed options (stringly-typed)
- Forgetting `ValidateOnStart()` -- validation only runs on first access otherwise
- Using `IOptionsSnapshot` in a singleton service (scoped lifetime conflict)
- Committing secrets in `appsettings.json` -- use environment variables or secret managers

## Related

- [strongly-typed-configuration.md](./strongly-typed-configuration.md) -- Advanced patterns
- [dependency-injection.md](./dependency-injection.md) -- Service registration
