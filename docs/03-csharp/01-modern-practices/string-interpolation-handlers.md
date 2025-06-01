# String Interpolation Handlers

Custom interpolated string handlers for high-performance string building. DefaultInterpolatedStringHandler, conditional logging, and creating custom handlers.

Keywords: interpolated string handler, DefaultInterpolatedStringHandler, InterpolatedStringHandlerAttribute, string.Format, logging, ILogger, performance, conditional string building

## Principle

Custom interpolated string handlers move string formatting cost from the call site to the handler, enabling conditional evaluation, reduced allocation, and compile-time optimization of interpolated strings.

## DefaultInterpolatedStringHandler

Since C# 10, the compiler uses `DefaultInterpolatedStringHandler` instead of `string.Format` for interpolated strings, producing fewer allocations.

```csharp
// What the compiler generates for: $"User {name} logged in at {time:HH:mm}"

// Old (pre-C# 10): string.Format
string result1 = string.Format("User {0} logged in at {1:HH:mm}", name, time);
// Allocates: format string parsing, object[] for args, boxed DateTime

// New (C# 10+): DefaultInterpolatedStringHandler
var handler = new DefaultInterpolatedStringHandler(20, 2);
handler.AppendLiteral("User ");
handler.AppendFormatted(name);
handler.AppendLiteral(" logged in at ");
handler.AppendFormatted(time, "HH:mm");
string result2 = handler.ToStringAndClear();
// No object[], no boxing, writes directly to internal buffer
```

## Performance Benefits Over string.Format

```csharp
// Benchmark: Format a string with 3 parameters, 100,000 iterations

// | Method                | Mean    | Allocated |
// |-----------------------|---------|-----------|
// | string.Format         | 180 us  | 4.8 MB    |
// | $"" (handler)         | 95 us   | 2.4 MB    |
// | StringBuilder         | 110 us  | 3.1 MB    |
// | string.Create + Span  | 60 us   | 1.6 MB    |

// string.Format boxes value types and allocates params array
string bad = string.Format("Order {0}: {1} items, total ${2:F2}", orderId, count, total);

// Interpolated string uses handler - no boxing, no params array
string good = $"Order {orderId}: {count} items, total ${total:F2}";
```

## Conditional String Building (Logging Optimization)

The key advantage of custom handlers: the string is only built if the handler decides to proceed.

```csharp
// Problem: string is always built, even when log level is disabled
logger.LogDebug($"Processing order {orderId} with {items.Count} items for {customer.Name}");
// The interpolated string is evaluated BEFORE LogDebug checks if Debug is enabled

// Solution with custom handler: string is only built when needed
// Microsoft.Extensions.Logging uses this pattern internally
public static partial class LogMessages
{
    [LoggerMessage(Level = LogLevel.Debug, Message = "Processing order {OrderId} with {ItemCount} items for {CustomerName}")]
    public static partial void LogProcessingOrder(
        this ILogger logger, string orderId, int itemCount, string customerName);
}

// Usage - zero cost when Debug is disabled
logger.LogProcessingOrder(orderId, items.Count, customer.Name);
```

## Creating Custom Handlers

Build a custom handler using `InterpolatedStringHandlerAttribute` and the `AppendLiteral`/`AppendFormatted` pattern.

```csharp
// A handler that conditionally builds strings based on a flag
[InterpolatedStringHandler]
public ref struct ConditionalStringHandler
{
    private DefaultInterpolatedStringHandler _inner;
    private readonly bool _enabled;

    public ConditionalStringHandler(
        int literalLength,
        int formattedCount,
        bool condition,
        out bool handlerIsValid)
    {
        _enabled = condition;
        handlerIsValid = condition; // If false, compiler skips all Append calls

        if (_enabled)
            _inner = new DefaultInterpolatedStringHandler(literalLength, formattedCount);
    }

    public void AppendLiteral(string s)
    {
        if (_enabled) _inner.AppendLiteral(s);
    }

    public void AppendFormatted<T>(T value)
    {
        if (_enabled) _inner.AppendFormatted(value);
    }

    public void AppendFormatted<T>(T value, string? format)
    {
        if (_enabled) _inner.AppendFormatted(value, format);
    }

    public string ToStringAndClear() =>
        _enabled ? _inner.ToStringAndClear() : string.Empty;
}
```

Using the custom handler:

```csharp
public static class ConditionalWriter
{
    public static void WriteIf(
        bool condition,
        [InterpolatedStringHandlerArgument("condition")] ConditionalStringHandler message)
    {
        if (condition)
            Console.WriteLine(message.ToStringAndClear());
    }
}

// Usage - string is never built when condition is false
bool verbose = false;
ConditionalWriter.WriteIf(verbose, $"Detailed info: {ExpensiveComputation()}");
// ExpensiveComputation() is never called when verbose is false
```

## AppendLiteral / AppendFormatted Pattern

The compiler calls these methods in sequence for each part of the interpolated string.

```csharp
// For: $"Hello {name}, you have {count} messages"
// Compiler generates:
handler.AppendLiteral("Hello ");
handler.AppendFormatted(name);
handler.AppendLiteral(", you have ");
handler.AppendFormatted(count);
handler.AppendLiteral(" messages");

// AppendFormatted overloads handle different types and formats:
public void AppendFormatted<T>(T value);                              // Generic
public void AppendFormatted<T>(T value, string? format);              // With format string
public void AppendFormatted<T>(T value, int alignment);               // With alignment
public void AppendFormatted<T>(T value, int alignment, string? format); // Both
public void AppendFormatted(ReadOnlySpan<char> value);                // Span<char>
public void AppendFormatted(string? value);                           // String (optimized)
```

## Compile-Time Handler Generation

Source generators can produce handlers that are fully optimized at compile time.

```csharp
// LoggerMessage source generator produces optimized code
public static partial class Log
{
    [LoggerMessage(
        EventId = 1001,
        Level = LogLevel.Information,
        Message = "Order {OrderId} created for customer {CustomerId} with {ItemCount} items")]
    public static partial void OrderCreated(
        this ILogger logger, string orderId, string customerId, int itemCount);
}

// Generated code (simplified) avoids interpolation entirely:
// - Checks IsEnabled(LogLevel.Information) first
// - Uses a cached LogDefineMessage delegate
// - No string allocation when logging is disabled
// - No boxing of value types
```

## ILogger High-Performance Logging

Combine interpolation handlers with ILogger for zero-cost-when-disabled logging.

```csharp
// BAD: Always allocates the interpolated string
logger.LogDebug($"User {userId} accessed {resource} at {DateTime.UtcNow}");

// BAD: Always evaluates arguments (ToString, property access)
logger.LogDebug("User {UserId} accessed {Resource}", userId.ToString(), resource.Path);

// GOOD: Source-generated, zero-cost when disabled
public static partial class LogMessages
{
    [LoggerMessage(Level = LogLevel.Debug,
        Message = "User {UserId} accessed {Resource}")]
    public static partial void UserAccessed(this ILogger logger, Guid userId, string resource);

    [LoggerMessage(Level = LogLevel.Warning,
        Message = "Rate limit exceeded for {ClientIp}: {RequestCount} requests in {WindowSeconds}s")]
    public static partial void RateLimitExceeded(
        this ILogger logger, string clientIp, int requestCount, int windowSeconds);

    [LoggerMessage(Level = LogLevel.Error,
        Message = "Failed to process order {OrderId}")]
    public static partial void OrderProcessingFailed(
        this ILogger logger, string orderId, Exception exception);
}

// Usage
logger.UserAccessed(userId, "/api/orders");
logger.RateLimitExceeded(clientIp, requestCount, 60);
logger.OrderProcessingFailed(orderId, ex);
```

## Building a Debug-Only Handler

A handler that only builds strings in debug builds:

```csharp
[InterpolatedStringHandler]
public ref struct DebugStringHandler
{
    private DefaultInterpolatedStringHandler _inner;
    private readonly bool _enabled;

    public DebugStringHandler(int literalLength, int formattedCount, out bool handlerIsValid)
    {
#if DEBUG
        _enabled = true;
        handlerIsValid = true;
        _inner = new DefaultInterpolatedStringHandler(literalLength, formattedCount);
#else
        _enabled = false;
        handlerIsValid = false;
        _inner = default;
#endif
    }

    public void AppendLiteral(string s) { if (_enabled) _inner.AppendLiteral(s); }
    public void AppendFormatted<T>(T value) { if (_enabled) _inner.AppendFormatted(value); }

    public override string ToString() => _enabled ? _inner.ToStringAndClear() : string.Empty;
}

public static class DebugLog
{
    public static void Write(DebugStringHandler message)
    {
#if DEBUG
        System.Diagnostics.Debug.WriteLine(message.ToString());
#endif
    }
}

// Usage - completely eliminated in Release builds
DebugLog.Write($"Cache stats: {cache.Count} entries, {cache.HitRate:P2} hit rate");
```

## Best Practices

**DO:**
- Use `[LoggerMessage]` source generator for all structured logging
- Let the compiler use DefaultInterpolatedStringHandler (prefer `$""` over `string.Format`)
- Use custom handlers when string building should be conditional
- Pass `out bool handlerIsValid` to short-circuit expensive operations

**DON'T:**
- Use `$""` interpolation in hot-path logging without source generation
- Create custom handlers for simple unconditional string building
- Forget the `[InterpolatedStringHandlerArgument]` attribute when linking handler to a method parameter
- Use `string.Format` or `string.Concat` when interpolation is available

## Guidelines

**Essential:**
- Prefer `$""` over `string.Format` everywhere (better codegen since C# 10)
- Use `[LoggerMessage]` for all ILogger calls in production code
- Understand that interpolated strings in disabled log levels still allocate without source generation

**Recommended:**
- Build custom handlers for conditional output (debug logging, verbose mode, feature flags)
- Use `InterpolatedStringHandlerArgument` to pass context (log level, condition) to the handler
- Combine handlers with Span-based formatting for zero-allocation output

**Advanced:**
- Create domain-specific handlers (SQL builders, URL formatters, template engines)
- Use ref struct handlers for stack-only lifetime and maximum performance
- Integrate custom handlers with source generators for compile-time optimization

## Benefits

Conditional evaluation. Expensive arguments are never evaluated when the handler short-circuits.

Reduced allocations. DefaultInterpolatedStringHandler avoids boxing and params arrays.

Source generator integration. LoggerMessage eliminates all formatting cost when log level is disabled.

Extensibility. Custom handlers enable domain-specific optimized string building patterns.

## Related

- [raw-string-literals.md](./raw-string-literals.md) - Raw string literal syntax
- [span-memory.md](./span-memory.md) - Span-based formatting and parsing
