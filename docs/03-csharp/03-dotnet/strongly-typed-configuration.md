# Strongly-Typed Configuration

Advanced configuration patterns: validation, hierarchical options, reloading, and secrets management.

## Why It Matters

- Invalid configuration fails fast at startup, not at runtime under load
- Hierarchical organization mirrors feature/service boundaries
- Options interfaces control reload behavior precisely

## Hierarchical Options

```csharp
public class ServiceOptions
{
    public const string SectionName = "Services";
    public DatabaseOptions Database { get; set; } = new();
    public CacheOptions Cache { get; set; } = new();
}

builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection("Services:Database"));
```

## Validation with DataAnnotations

```csharp
public class ApiOptions
{
    [Required, Url]
    public string BaseUrl { get; set; } = string.Empty;
    [Range(1, 300)]
    public int TimeoutSeconds { get; set; } = 30;
}

builder.Services.AddOptions<ApiOptions>()
    .Bind(builder.Configuration.GetSection("Api"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

## Custom Validation (IValidateOptions)

```csharp
public class SecurityOptionsValidator : IValidateOptions<SecurityOptions>
{
    public ValidateOptionsResult Validate(string? name, SecurityOptions options)
    {
        if (options.JwtSecret.Length < 32)
            return ValidateOptionsResult.Fail("JWT secret must be at least 32 characters");
        return ValidateOptionsResult.Success;
    }
}

builder.Services.AddSingleton<IValidateOptions<SecurityOptions>, SecurityOptionsValidator>();
```

## Options Interfaces Quick Reference

| Interface | Lifetime | Reloads | Best For |
|---|---|---|---|
| `IOptions<T>` | Singleton | Never | Static config, singleton services |
| `IOptionsSnapshot<T>` | Scoped | Per request | Config that changes occasionally |
| `IOptionsMonitor<T>` | Singleton | Real-time | Feature flags, `OnChange` callback |

## Environment-Specific Overrides

```
appsettings.json               -> base values
appsettings.Development.json   -> dev overrides
Environment variables          -> highest priority (use __ for hierarchy)
```

## Secrets Management

```csharp
// Never commit secrets. Use environment variables:
// ExternalService__ApiKey=secret_key_here

// Or AWS Secrets Manager:
builder.Configuration.AddSecretsManager();
```

## Post-Configure (Normalize After Binding)

```csharp
builder.Services.PostConfigure<DatabaseOptions>(opts =>
{
    opts.TablePrefix = opts.TablePrefix.ToUpperInvariant();
});
```

## Pitfalls to Avoid

- Accessing `Configuration["Section:Key"]` strings instead of typed options
- Forgetting `ValidateOnStart()` (validation defers to first access)
- Committing secrets in appsettings files
- Using `IOptionsSnapshot` in singleton services (lifetime mismatch)

## Related

- [configuration-options-pattern.md](./configuration-options-pattern.md) -- Basic options pattern
- [dependency-injection.md](./dependency-injection.md) -- Options injection
