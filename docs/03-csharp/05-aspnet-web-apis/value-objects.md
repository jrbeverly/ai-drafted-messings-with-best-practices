# Value Objects

Strongly-typed domain primitives that bind automatically in minimal APIs via `TryParse`.

## Why It Matters

- Type safety: can't accidentally pass `LibraryId` where `BookId` is expected
- Validation centralized in one place -- enforced everywhere the type is used
- Self-documenting: `Email email` is clearer than `string email`

## Pattern

```csharp
public record Email
{
    public string Value { get; }
    private Email(string value) => Value = value;

    public static bool TryParse(string? input, out Email? result)
    {
        if (string.IsNullOrWhiteSpace(input) || !input.Contains('@'))
        { result = null; return false; }

        result = new Email(input);
        return true;
    }

    public override string ToString() => Value;
}
```

## Automatic Binding

The framework calls `TryParse` for route and query parameters. Returns 400 if parsing fails.

```csharp
public static async Task<IResult> HandleAsync([FromRoute] Email email) { ... }
```

## Common Value Objects

- **Strongly-typed IDs:** `LibraryId` (prefix `lib_`), `BookId` (prefix `book_`)
- **Contact info:** `Email`, `PhoneNumber`
- **Domain values:** `Money` (amount + currency), `Percentage` (0-100)

## Usage in Request DTOs

```csharp
public record Request
{
    [Required] public Email Email { get; init; } = null!;
    [Required] public LibraryId LibraryId { get; init; } = null!;
    public PhoneNumber? PhoneNumber { get; init; }
}
```

## JSON Serialization

Register a `JsonConverter<T>` that delegates to `TryParse` for reading and `Value` for writing:

```csharp
builder.Services.Configure<JsonOptions>(o =>
    o.JsonSerializerOptions.Converters.Add(new EmailJsonConverter()));
```

## Key Recommendations

- Private constructor, public `TryParse`, immutable (`record`)
- Always override `ToString()` to return the inner value
- Name after the domain concept (`Email`, `Money`), not the technical wrapper (`EmailString`)
- Validate inside `TryParse`; return false for invalid input

## When to Use vs. Not

| Use | Skip |
|-----|------|
| Domain primitives with validation rules | Simple pass-through values |
| Strongly-typed IDs | Values without any validation |
| Values with behavior (formatting, comparison) | Performance-critical hot paths |

## Pitfalls to Avoid

- Public constructors that bypass validation
- Forgetting the `JsonConverter` (serialization will fail or produce `{}`)
- Over-wrapping trivial primitives that have no validation rules

## Related

- [parameter-binding.md](./parameter-binding.md) -- TryParse binding mechanics
- [request-response-pattern.md](./request-response-pattern.md) -- using in DTOs
