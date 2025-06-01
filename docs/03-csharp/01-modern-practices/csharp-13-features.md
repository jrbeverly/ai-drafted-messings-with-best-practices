# C# 13 Features

Language features introduced with C# 13 / .NET 9: params collections, Lock type, escape sequence, allows ref struct, partial properties, and overload resolution priority.

## Why It Matters

- `params ReadOnlySpan<T>` eliminates array allocation on every variadic call
- `System.Threading.Lock` replaces `lock(object)` with clearer intent and better perf
- `allows ref struct` opens generics to Span-based types

## Key Features

**Params collections** -- `params` works with any collection type:
```csharp
public int Sum(params ReadOnlySpan<int> values) { /* zero allocation */ }
Sum(1, 2, 3); // no int[] created
```

**New Lock type** (`System.Threading.Lock`):
```csharp
private readonly Lock _lock = new();
lock (_lock) { /* critical section */ }
```
Not async-safe -- use `SemaphoreSlim` for async critical sections.

**`\e` escape sequence** for ANSI escape (replaces `\u001b` / `\x1b`):
```csharp
string red = "\e[31mError\e[0m";
```

**Allows ref struct** generic constraint:
```csharp
public static T Sum<T>(params ReadOnlySpan<T> values) where T : INumber<T>, allows ref struct
```

**Partial properties** -- source generators declare, developers implement:
```csharp
public partial class VM { public partial string Name { get; set; } }
```

**Overload resolution priority** -- library evolution without breaking changes:
```csharp
[OverloadResolutionPriority(1)]
public static string Serialize<T>(T value, JsonSerializerOptions? opts = null) => /* ... */;
```

**Implicit indexer `[^n]` in object initializers** and **ref/unsafe in iterators and async** (with restrictions around yield/await boundaries).

## Pitfalls to Avoid

- Holding `ref` locals across `yield`/`await` boundaries (compiler error)
- Using `OverloadResolutionPriority` for general disambiguation (library evolution only)
- Replacing working `lock(object)` without measuring benefit
- Using partial properties without a source generator (unnecessary complexity)
