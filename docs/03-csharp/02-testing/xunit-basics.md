# xUnit Basics

The default test framework for .NET -- write, organize, and run unit tests.

## Why It Matters

- Industry standard for .NET testing with strong tooling support
- Constructor-per-test isolation prevents shared state bugs
- Native async/await and parameterized test support

## Installation

```bash
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package Microsoft.NET.Test.Sdk
```

## Fact vs Theory

```csharp
[Fact]   // single scenario
public void Add_TwoNumbers_ReturnsSum()
{
    Assert.Equal(5, new Calculator().Add(2, 3));
}

[Theory]  // parameterized
[InlineData(2, 3, 5)]
[InlineData(-1, 1, 0)]
public void Add_MultipleInputs_ReturnsCorrectSum(int a, int b, int expected)
{
    Assert.Equal(expected, new Calculator().Add(a, b));
}
```

## Common Assertions

```csharp
Assert.Equal(expected, actual);          Assert.NotEqual(a, b);
Assert.True(condition);                  Assert.False(condition);
Assert.Null(value);                      Assert.NotNull(value);
Assert.IsType<User>(result);             Assert.Contains(item, collection);
Assert.Empty(collection);               Assert.InRange(actual, low, high);
Assert.Throws<ArgumentException>(() => method());
await Assert.ThrowsAsync<InvalidOperationException>(() => asyncMethod());
```

## Test Data Sources

- **`[InlineData]`** -- simple inline values
- **`[MemberData(nameof(Prop))]`** -- static property returning `IEnumerable<object[]>`
- **`[ClassData(typeof(T))]`** -- reusable class implementing `IEnumerable<object[]>`

## Setup and Teardown

- **Constructor** -- runs before each test (fresh instance per test)
- **`IDisposable`** -- cleanup after each test
- **`IAsyncLifetime`** -- async `InitializeAsync` / `DisposeAsync`
- **`IClassFixture<T>`** -- shared setup across all tests in a class

## Running Tests

```bash
dotnet test                                        # all tests
dotnet test --filter "ClassName=UserServiceTests"  # by class
dotnet test --filter "Category=Unit"               # by trait
```

## Test Organization

Use nested classes to group related tests:

```csharp
public class UserServiceTests
{
    public class GetUser
    {
        [Fact] public void ValidId_ReturnsUser() { }
        [Fact] public void InvalidId_ThrowsNotFoundException() { }
    }
}
```

## Pitfalls to Avoid

- Sharing mutable static state across tests (constructor runs per-test for isolation)
- Forgetting `async Task` return type on async test methods
- Using `[Fact]` when `[Theory]` with `[InlineData]` would reduce duplication
- Skipping `Microsoft.NET.Test.Sdk` package (tests will not be discovered)

## Related

- [unit-testing-best-practices.md](./unit-testing-best-practices.md) -- Testing patterns
- [mocking-with-nsubstitute.md](./mocking-with-nsubstitute.md) -- Mocking dependencies
