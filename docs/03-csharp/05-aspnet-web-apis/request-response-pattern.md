# Request/Response Pattern

Dedicated DTOs for each endpoint's request and response. No sharing between endpoints or versions.

## Why It Matters

- Independent evolution: changing one endpoint never breaks another
- Version-specific types enable API evolution without breaking changes
- Each endpoint optimizes its contract for its specific use case

## Structure

Nested records inside the endpoint class:

```csharp
public static class LoanCreateRoute
{
    public record Request : IValidatableObject
    {
        [Required] public string BookId { get; init; } = string.Empty;
        [Range(1, 90)] public int DurationDays { get; init; } = 14;

        public IEnumerable<ValidationResult> Validate(ValidationContext ctx)
        {
            yield break; // cross-property rules here
        }
    }

    public record Response
    {
        public string LoanId { get; init; } = string.Empty;
        public DateTime DueDate { get; init; }
    }
}
```

## DataAnnotations Quick Reference

```
[Required]  [EmailAddress]  [Phone]  [Url]  [CreditCard]
[Range(1, 100)]  [MinLength(2)]  [MaxLength(100)]
[RegularExpression(@"^book_[A-Z0-9]+$")]
[Compare("Password")]
```

Custom error messages: `[Required(ErrorMessage = "Email is required")]`

## Cross-Property Validation

Implement `IValidatableObject` for rules spanning multiple fields:

```csharp
if (EndDate <= StartDate)
    yield return new ValidationResult("End date must be after start date", [nameof(EndDate)]);
```

## Version-Specific DTOs

```csharp
namespace Routes.Loans.v1;
public record Response { public string LoanId { get; init; } }

namespace Routes.Loans.v2;
public record Response { public string LoanId { get; init; } public string BookTitle { get; init; } }
```

## Key Recommendations

- **Never share** Request/Response types between endpoints, even if they look similar
- Use `init`-only properties with defaults for immutability
- Map DTOs to domain models inside the handler -- never expose domain models directly
- `.NET 10+`: `builder.Services.AddValidation();` enables automatic validation

## Pitfalls to Avoid

- Reusing a "shared" DTO across create/update/get endpoints
- Exposing domain entities directly as responses
- Skipping `IValidatableObject` when cross-property rules exist
- Mutable `set` properties instead of `init`

## Related

- [endpoint-organization.md](./endpoint-organization.md) -- full endpoint structure
- [endpoint-filters.md](./endpoint-filters.md) -- validation filters
