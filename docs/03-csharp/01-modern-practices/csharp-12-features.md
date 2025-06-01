# C# 12 Features

Language features introduced with C# 12 / .NET 8: primary constructors, collection expressions, type aliases, default lambda parameters, inline arrays, and ref readonly parameters.

## Why It Matters

- Primary constructors eliminate DI boilerplate on classes and structs
- Collection expressions unify initialization syntax
- Type aliases clarify complex tuple and generic types

## Key Features

**Primary constructors** (classes and structs):
```csharp
public class OrderService(IOrderRepository repo, ILogger<OrderService> logger)
{
    public async Task<Order> GetAsync(Guid id) => await repo.GetByIdAsync(id);
}
```
Capture to `readonly` field when parameter is used in many methods or needs readonly semantics.

**Collection expressions** -- see [collection-expressions.md](./collection-expressions.md):
```csharp
int[] nums = [1, 2, 3];
int[] all = [..first, ..second];
```

**Alias any type** with `using`:
```csharp
using Point = (double X, double Y);
using StringMap = System.Collections.Generic.Dictionary<string, string>;
```

**Default lambda parameters:**
```csharp
var greet = (string name, string greeting = "Hello") => $"{greeting}, {name}!";
```

**Inline arrays** -- fixed-size, heap-free buffers (library/runtime authors):
```csharp
[InlineArray(4)]
public struct FourInts { private int _e0; }
```

**Ref readonly parameters** -- see [ref-readonly-parameters.md](./ref-readonly-parameters.md).

## Pitfalls to Avoid

- Using interceptors in application code (experimental, source-generator only)
- Aliasing already-clear types (`using Name = string`)
- Inline arrays in non-performance-critical code (adds complexity)
