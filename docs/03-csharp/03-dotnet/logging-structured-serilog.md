# Structured Logging with Serilog

Production-grade structured logging with rich context, multiple sinks, and powerful enrichment.

## Why It Matters

- Logs are structured objects (not just strings), enabling powerful queries
- Multiple sinks write to Console, File, CloudWatch, Seq simultaneously
- Automatic enrichment adds machine, thread, correlation, and custom properties

## Installation

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
```

## Minimal Setup

```csharp
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .CreateLogger();

builder.Host.UseSerilog();
```

## appsettings.json Configuration

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": { "Microsoft": "Warning", "System": "Warning" }
    },
    "Enrich": ["FromLogContext", "WithMachineName", "WithThreadId"]
  }
}
```

## Structured Properties

```csharp
// Named placeholders -- searchable in log aggregators
_logger.LogInformation("User {UserId} updated email to {Email}", userId, email);

// Destructure objects with @ prefix
_logger.LogInformation("Created user {@User}", user);
```

## Log Context (Scoped Properties)

```csharp
using (LogContext.PushProperty("CorrelationId", correlationId))
{
    // All logs in this scope include CorrelationId automatically
}
```

## Request Logging Middleware

```csharp
app.UseSerilogRequestLogging(opts =>
{
    opts.EnrichDiagnosticContext = (ctx, httpCtx) =>
    {
        ctx.Set("UserAgent", httpCtx.Request.Headers["User-Agent"].ToString());
    };
});
```

## Common Sinks

| Sink | Package | Use For |
|---|---|---|
| Console | `Serilog.Sinks.Console` | Development |
| File | `Serilog.Sinks.File` | Simple production |
| CloudWatch | `Serilog.Sinks.AwsCloudWatch` | AWS deployments |
| Seq | `Serilog.Sinks.Seq` | Local structured log viewer |

## Key Recommendations

- Use `ReadFrom.Configuration()` to control levels without redeployment
- Add `Enrich.FromLogContext()` for scoped property support
- Use `CompactJsonFormatter` for machine-readable production output
- Wrap `app.Run()` in try/catch with `Log.Fatal` and `Log.CloseAndFlush()`

## Pitfalls to Avoid

- String interpolation in log messages (use named placeholders)
- Logging PII or secrets (passwords, tokens, connection strings)
- Forgetting `Log.CloseAndFlush()` -- buffered sinks may lose final entries
- Over-enriching with large objects (increases log size and cost)

## Related

- [logging-ilogger.md](./logging-ilogger.md) -- Built-in ILogger patterns
- [middleware-aspnet.md](./middleware-aspnet.md) -- Request logging middleware
