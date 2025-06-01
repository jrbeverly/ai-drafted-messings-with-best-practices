# Init-Only Properties

C# 9 `init` accessor that allows properties to be set during object initialization but prevents modification afterward.

## Why It Matters

- Enables immutable objects with flexible object-initializer syntax
- No constructor boilerplate needed for simple immutable types
- Thread-safe by construction (immutable after creation)

## Key Recommendations

**Use `init` instead of `set` for immutable data:**
```csharp
public class User
{
    public string Name { get; init; } = string.Empty;
    public string Email { get; init; } = string.Empty;
}

var user = new User { Name = "John", Email = "john@example.com" };
user.Name = "Jane"; // Compiler error
```

**Combine with `required` for mandatory properties** (C# 11+):
```csharp
public required string Id { get; init; }
```

**Provide default values for optional properties:**
```csharp
public bool IsPublic { get; init; } = true;
public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
```

**Use `with` expressions to create modified copies** (records or classes):
```csharp
var updated = user with { Age = 31 }; // new instance, original unchanged
```

**Use `IReadOnlyList<T>` for collection properties** to achieve true deep immutability.

## When to Use

- API request/response DTOs
- Configuration and options objects
- Domain entities and value objects
- Any data object that should not change after creation

## Pitfalls to Avoid

- Using `init` when properties legitimately need to change (state machines, mutable entities)
- Assuming collection properties are deeply immutable (`IReadOnlyList` prevents reassignment but not mutation of mutable inner items)
