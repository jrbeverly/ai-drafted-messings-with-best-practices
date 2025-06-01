# Unit Testing Best Practices

Patterns and practices for writing fast, focused, and maintainable unit tests.

## Why It Matters

- Fast feedback loop catches bugs during development, not in production
- Tests document how code should be used
- Hard-to-test code is a signal of poor design

## Core Rules

- **One logical assertion per test** -- test a single behavior/scenario
- **Arrange-Act-Assert** -- always follow this three-phase structure
- **Test public API, not internals** -- assert on observable behavior
- **No shared mutable state** -- each test creates its own data
- **No I/O** -- unit tests should run in-memory, under 100ms

## Naming Convention

```
MethodName_Scenario_ExpectedResult
```

```csharp
GetUser_NonExistentId_ThrowsNotFoundException()
ValidateEmail_EmptyString_ReturnsFalse()
CalculateDiscount_PremiumMember_Returns20Percent()
```

## Use Theory for Multiple Inputs

```csharp
[Theory]
[InlineData(10, 20, 30)]
[InlineData(0, 0, 0)]
[InlineData(-1, 1, 0)]
public void Add_TwoNumbers_ReturnsSum(int a, int b, int expected)
{
    Assert.Equal(expected, new Calculator().Add(a, b));
}
```

## Assert Exceptions Properly

```csharp
var ex = Assert.Throws<ArgumentNullException>(() => service.GetUser(null!));
Assert.Equal("id", ex.ParamName);
```

## Always Cover

- Happy path
- Edge cases (null, empty, boundary values)
- Error conditions (exceptions, invalid input)

## Pitfalls to Avoid

- **Testing multiple behaviors in one test** -- failures are ambiguous
- **Logic in tests** (loops, conditionals) -- tests should be linear
- **Shared static state** between tests -- leads to order-dependent failures
- **Backwards `Assert.Equal`** -- first param is `expected`, second is `actual`
- **Testing private methods** -- test through the public API instead

## Related

- [xunit-basics.md](./xunit-basics.md) -- xUnit fundamentals
- [mocking-with-nsubstitute.md](./mocking-with-nsubstitute.md) -- Isolating dependencies
- [test-data-builders.md](./test-data-builders.md) -- Readable test data
