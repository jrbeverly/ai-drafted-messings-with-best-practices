# Default Interface Methods

C# 8+ feature allowing interfaces to provide method implementations, enabling interface evolution without breaking existing implementations.

## Why It Matters

- Add new methods to interfaces without breaking all implementors
- Provide sensible default behavior that can be overridden
- Bridge between interfaces (no state) and abstract classes (shared behavior)

## Key Recommendations

**Evolve interfaces by adding methods with defaults:**
```csharp
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(string id);
    Task SaveAsync(T entity);

    // Added in v2 -- existing implementations still compile
    async Task<bool> ExistsAsync(string id)
        => await GetByIdAsync(id) is not null;
}
```

**Provide convenience methods on interfaces:**
```csharp
public interface ILogger
{
    void Log(LogLevel level, string message);
    void LogError(string msg) => Log(LogLevel.Error, $"ERROR: {msg}");
    void LogInfo(string msg) => Log(LogLevel.Information, msg);
}
```

**Override defaults when a more efficient implementation exists.**

**Resolve conflicts with explicit interface implementation** when a class implements two interfaces with the same default method.

**Default methods are only accessible through the interface type**, not the concrete class reference.

## When to Use

- Interface evolution (adding methods to published interfaces)
- Convenience/helper methods derived from core methods
- Optional functionality with fallback behavior

## Pitfalls to Avoid

- Complex logic in default methods (keep them simple)
- State in interfaces (use abstract classes for shared state)
- Replacing abstract classes entirely (different purposes)
- Assuming defaults are available on the concrete type (must cast to interface)
