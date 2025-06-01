# Test Data Builders

Builder pattern for constructing test objects with sensible defaults and fluent overrides.

## Why It Matters

- Tests only specify what is relevant to the scenario, improving readability
- Domain model changes require updating the builder, not every test
- Encourages DRY test setup without sacrificing clarity

## Basic Builder

```csharp
public class UserBuilder
{
    private string _id = "user_123";
    private string _email = "test@example.com";
    private string _name = "Test User";
    private bool _isActive = true;

    public UserBuilder WithId(string id) { _id = id; return this; }
    public UserBuilder WithEmail(string email) { _email = email; return this; }
    public UserBuilder WithName(string name) { _name = name; return this; }
    public UserBuilder Inactive() { _isActive = false; return this; }

    public User Build() => new() { Id = _id, Email = _email, Name = _name, IsActive = _isActive };
}
```

## Usage in Tests

```csharp
// Only specify what matters for this test
var user = new UserBuilder().WithEmail("john@example.com").Build();
var book = new BookBuilder().Unavailable().Build();
```

## Object Mother (Predefined Scenarios)

```csharp
public static class TestUsers
{
    public static User Default() => new UserBuilder().Build();
    public static User Admin() => new UserBuilder().WithEmail("admin@example.com").Build();
    public static User Inactive() => new UserBuilder().Inactive().Build();
}
```

## Composite Builders (Related Objects)

```csharp
var (user, book, loan) = new LoanScenarioBuilder()
    .WithBook(b => b.Unavailable())
    .Build();  // Automatically wires user/book IDs into the loan
```

## When to Use

- Complex objects with many properties
- Same entity appears across many test scenarios
- Domain model changes frequently

## When NOT to Use

- Simple objects with 1-2 properties (use object initializers)
- One-off test data that does not repeat

## Pitfalls to Avoid

- Forgetting sensible defaults (every `Build()` should produce a valid object)
- Builder methods with unclear names -- prefer intent-revealing names like `Overdue()`, `Inactive()`
- Over-engineering builders for trivial models

## Related

- [xunit-basics.md](./xunit-basics.md) -- Test fundamentals
- [unit-testing-best-practices.md](./unit-testing-best-practices.md) -- Testing patterns
