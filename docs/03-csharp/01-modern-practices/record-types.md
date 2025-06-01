# Record Types

Immutable reference types with built-in value equality, deconstruction, and `with` expression support (C# 9+).

## Why It Matters

- Value equality without manual `Equals`/`GetHashCode` implementations
- Immutable by default -- prevents accidental mutation
- Concise positional syntax for DTOs and value objects

## Key Recommendations

**Positional records** for compact declarations:
```csharp
public record User(string Name, string Email);
var u1 = new User("John", "john@example.com");
var (name, email) = u1; // deconstruction
```

**Property records** for DTOs with `required`/`init`:
```csharp
public record CreateUserRequest
{
    public required string Name { get; init; }
    public required string Email { get; init; }
    public string? Phone { get; init; }
}
```

**Value equality** is automatic:
```csharp
var a = new User("John", "john@example.com");
var b = new User("John", "john@example.com");
a == b // true; works in HashSet/Dictionary
```

**`with` expressions** for non-destructive mutation:
```csharp
var updated = user with { Email = "new@example.com" };
```

**`record struct`** for value-type records (stack-allocated):
```csharp
public record struct Point(int X, int Y);
```

**Inheritance** works with records:
```csharp
public record Person(string Name);
public record Employee(string Name, string Dept) : Person(Name);
```

**Domain value objects** with validation:
```csharp
public record Money(decimal Amount, string Currency)
{
    public Money Add(Money other) => Currency == other.Currency
        ? this with { Amount = Amount + other.Amount }
        : throw new InvalidOperationException("Currency mismatch");
}
```

## Record vs Class

| Use Record | Use Class |
|------------|-----------|
| Value equality needed | Identity equality needed |
| Immutable data (DTOs, value objects) | Mutable state, lifecycle entities |
| Data transfer across boundaries | Large object graphs |

## Pitfalls to Avoid

- Using records for entities where identity matters (same data != same entity)
- Forgetting that `ToString()` prints all properties (may leak sensitive data in logs)
- Deep mutation on nested reference properties is not blocked by `init`
