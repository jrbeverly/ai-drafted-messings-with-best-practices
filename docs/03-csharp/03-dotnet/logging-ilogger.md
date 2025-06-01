# Logging with ILogger

Structured logging with the built-in .NET `ILogger<T>` abstraction.

## Why It Matters

- Observability: understand application behavior in production
- Structured data enables filtering and querying across log aggregators
- Consistent abstraction works with any provider (Console, Serilog, CloudWatch)

## Basic Usage

```csharp
public class UserService(IUserRepository repository, ILogger<UserService> logger) : IUserService
{
    public async Task<User> GetUserAsync(Guid id)
    {
        logger.LogInformation("Retrieving user {UserId}", id);
        var user = await repository.GetByIdAsync(id);
        if (user is null)
        {
            logger.LogWarning("User {UserId} not found", id);
            throw new NotFoundException("User", id.ToString());
        }
        return user;
    }
}
```

## Structured Logging (Critical Rule)

```csharp
// CORRECT -- structured, searchable properties
_logger.LogInformation("User {UserId} updated email to {Email}", userId, email);

// WRONG -- string interpolation, not searchable
_logger.LogInformation($"User {userId} updated email to {email}");
```

## Log Levels

| Level | Use For |
|---|---|
| Trace/Debug | Development-only detail |
| Information | Normal flow, significant events |
| Warning | Unexpected but recoverable issues |
| Error | Handled failures with exception context |
| Critical | System failures requiring immediate attention |

## High-Performance Logging (Source Generators)

```csharp
public partial class UserService
{
    [LoggerMessage(EventId = 1, Level = LogLevel.Information,
        Message = "Retrieving user {UserId}")]
    private partial void LogRetrievingUser(Guid userId);
}
```

Lower allocation, compile-time validation of log message templates.

## Log Scopes (Contextual Properties)

```csharp
using (_logger.BeginScope(new Dictionary<string, object>
    { ["OrderId"] = orderId, ["UserId"] = userId }))
{
    _logger.LogInformation("Processing order");
    // All logs in this block include OrderId and UserId
}
```

## Configuration

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning"
    }
  }
}
```

## Key Recommendations

- Always use message template placeholders, never string interpolation
- Pass the exception object as first param: `LogError(ex, "Failed {Id}", id)`
- Use `LoggerMessage` source generators on hot paths
- Log at request boundaries, domain events, external calls, and errors

## Pitfalls to Avoid

- Logging sensitive data (passwords, tokens, PII)
- Logging inside tight loops (performance and noise)
- Using `LogDebug` in production without filtering (floods logs)
- String interpolation in log messages (defeats structured logging)

## Related

- [logging-structured-serilog.md](./logging-structured-serilog.md) -- Advanced Serilog patterns
