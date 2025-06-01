# Endpoint Organization

One file per endpoint, containing all concerns: registration, request/response DTOs, validation, handler, and OpenAPI examples.

## Why It Matters

- Self-contained: everything about an endpoint lives in one place
- Versioned namespaces allow independent evolution without breaking changes
- Adding endpoints never affects existing ones

## File & Folder Structure

```
Routes/{Resource}/v{N}/{Resource}{Action}Route.cs
```

```
Routes/
├── Loans/v1/LoanCreateRoute.cs
├── Loans/v1/LoanGetRoute.cs
├── Loans/v2/LoanCreateRoute.cs
└── Users/v1/UserGetRoute.cs
```

## Nested Class Layout

```csharp
namespace LibraryService.Routes.Loans.v1;

public static class LoanCreateRoute
{
    public const string ResourceName = "loan-create";
    public const int Version = 1;

    public static class Registration { /* Map(RouteGroupBuilder) */ }
    public static class Examples    { /* RequestExample, ResponseExample JSON */ }
    public record Request : IValidatableObject { /* DataAnnotations + Validate() */ }
    public record Response { /* output fields */ }
    public static class Handler { /* HandleAsync(...) */ }
}
```

## Wiring in Program.cs

```csharp
var loanGroup = v1.MapGroup("/loans/{libraryId}");
LoanCreateRoute.Registration.Map(loanGroup);
LoanGetRoute.Registration.Map(loanGroup);
```

## Key Recommendations

- **Naming:** File = `{Resource}{Action}Route.cs`, class = `{Resource}{Action}Route`, resource name = kebab-case
- **Versioning:** Major versions only (v1, v2). Independent implementations -- no shared code between versions
- **Validation:** DataAnnotations for simple rules; `IValidatableObject` for cross-property checks. `.NET 10+`: `builder.Services.AddValidation();`
- **Dependencies:** Inject services directly into `HandleAsync` parameters -- no base classes

## Pitfalls to Avoid

- Sharing Request/Response types between endpoints or versions
- Using inheritance or base classes for handlers
- Mixing multiple endpoints in one file
- Placing endpoint-specific logic in shared code

## Related

- [minimal-api-routing.md](./minimal-api-routing.md) -- route registration
- [request-response-pattern.md](./request-response-pattern.md) -- DTO details
- [policy-services.md](./policy-services.md) -- authorization in handlers
