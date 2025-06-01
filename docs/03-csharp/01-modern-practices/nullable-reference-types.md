# Nullable Reference Types

C# 8+ feature that makes all reference types non-nullable by default, catching null dereferences at compile time.

## Why It Matters

- Eliminates an entire class of `NullReferenceException` at compile time
- Makes nullability explicit in API contracts (callers know what can be null)
- Refactoring safety -- changing nullability forces updates at all call sites

## Key Recommendations

**Enable globally** in `.csproj`:
```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
</PropertyGroup>
```

**Annotate nullable references with `?`:**
```csharp
string name = "John";       // Cannot be null
string? optional = null;    // Explicitly nullable
```

**Use modern null checks:**
```csharp
if (value is not null) { /* safe */ }
var result = value ?? "default";
value ??= "fallback";
ArgumentNullException.ThrowIfNull(input);
```

**Initialize non-nullable properties** via constructor, `required`, or default value:
```csharp
public required string Email { get; init; }
public string Name { get; init; } = string.Empty;
```

**Return `T?` when a method can legitimately return null:**
```csharp
public User? FindById(int id) => _users.GetValueOrDefault(id);
```

**Migrate incrementally** with `#nullable enable` per file if needed.

## Pitfalls to Avoid

- Overusing null-forgiving operator `!` (defeats the purpose; document why when used)
- `#nullable disable` except during migration
- Forgetting that `List<string?>` (nullable elements) differs from `List<string>?` (nullable list)
