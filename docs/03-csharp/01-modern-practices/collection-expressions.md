# Collection Expressions

C# 12 unified syntax for creating arrays, lists, spans, and other collections using square brackets.

## Why It Matters

- Consistent, concise syntax across all collection types
- Spread operator (`..`) composes collections without LINQ
- `ReadOnlySpan<T>` targets get stack allocation (zero heap)

## Key Recommendations

**Use `[]` for all collection initialization** (C# 12+ / .NET 8+):
```csharp
int[] numbers = [1, 2, 3, 4, 5];
List<string> names = ["Alice", "Bob"];
ReadOnlySpan<int> span = [10, 20, 30]; // stack-allocated
int[] empty = [];
```

**Spread operator to combine collections:**
```csharp
int[] combined = [..first, ..second];
int[] extended = [0, ..first, 99];
User[] all = [..active, ..admins];
```

**Use in method parameters and returns:**
```csharp
ProcessNumbers([1, 2, 3]);
public string[] GetDefaultRoles() => ["User", "Guest"];
```

**Works with immutable collections:**
```csharp
ImmutableArray<int> values = [1, 2, 3];
```

**Default empty collection for record properties:**
```csharp
public string[] Roles { get; init; } = [];
```

## Pitfalls to Avoid

- Not supported for dictionaries (use traditional initializer syntax)
- Spread creates a new collection (copies all elements)
- `var` with `[]` infers array type; use explicit type for `List<T>` or `Span<T>`
