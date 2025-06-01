# Mocking with NSubstitute

Create test doubles for interfaces to isolate the unit under test from its dependencies.

## Why It Matters

- Tests run fast with no I/O or external systems
- Full control over dependency behavior, including error scenarios
- Verifies interactions between components without side effects

## Installation

```bash
dotnet add package NSubstitute
```

## Core API

```csharp
// Create a substitute
var repo = Substitute.For<IUserRepository>();

// Configure return values
repo.GetByIdAsync("user_123").Returns(new User { Id = "user_123", Name = "John" });
repo.GetByIdAsync(Arg.Any<string>()).Returns(new User { Name = "Fallback" });
repo.GetByIdAsync(Arg.Is<string>(id => id.StartsWith("user_"))).Returns(someUser);

// Return null
repo.GetByIdAsync("missing").Returns((User?)null);

// Throw exception
repo.GetByIdAsync("bad").Returns(Task.FromException<User?>(new DatabaseException("fail")));

// Sequential returns
repo.GetByIdAsync("x").Returns(first, second, third);
```

## Verifying Calls

```csharp
await repo.Received(1).SaveAsync(Arg.Any<User>());    // called exactly once
await repo.DidNotReceive().GetByIdAsync(Arg.Any<string>());  // never called
```

## Callbacks

```csharp
User? captured = null;
repo.SaveAsync(Arg.Any<User>())
    .Returns(Task.CompletedTask)
    .AndDoes(info => captured = info.Arg<User>());
```

## When to Mock

- External dependencies (databases, HTTP APIs, message queues)
- Slow or non-deterministic operations
- Side effects (email, notifications)

## Pitfalls to Avoid

- **Over-mocking:** mocking everything makes tests brittle and tightly coupled to implementation
- **Over-verifying:** assert on behavior/outcomes, not every internal method call
- **Mocking value objects or collections** (List, Dictionary) -- use the real thing
- **Mocking the class under test** -- only mock its dependencies
- Configuring returns you never use (adds noise, not signal)

## Related

- [xunit-basics.md](./xunit-basics.md) -- Test fundamentals
- [unit-testing-best-practices.md](./unit-testing-best-practices.md) -- Testing patterns
- [integration-testing.md](./integration-testing.md) -- When not to mock
