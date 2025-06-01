# ASP.NET Minimal API Best Practices

Opinionated guide for building minimal APIs with C# and .NET -- no controllers, MIME-type versioning, domain exceptions, and policy services.

## Core Principles

1. Minimal APIs only (no controllers)
2. MIME-type versioning via content negotiation
3. Versioned namespaces for independent evolution
4. DataAnnotations + IValidatableObject for validation (source-generator friendly)
5. Domain exceptions with Problem Details middleware
6. Policy services for authorization (fail-fast `Require*` methods)
7. Value objects with TryParse for type safety
8. JSON-only responses

## Documentation Map

| Topic | File | Key Idea |
|-------|------|----------|
| Endpoint structure | [endpoint-organization.md](./endpoint-organization.md) | One file per endpoint, nested static classes |
| Routing | [minimal-api-routing.md](./minimal-api-routing.md) | MapGroup hierarchy, Registration.Map() |
| Request/Response | [request-response-pattern.md](./request-response-pattern.md) | Dedicated DTOs per endpoint, no sharing |
| Filters | [endpoint-filters.md](./endpoint-filters.md) | IEndpointFilter for cross-cutting concerns |
| Domain exceptions | [domain-exceptions.md](./domain-exceptions.md) | Typed exceptions to Problem Details via middleware |
| Policy services | [policy-services.md](./policy-services.md) | Centralized auth with Require* methods |
| Error responses | [api-error-responses.md](./api-error-responses.md) | RFC 7807 Problem Details |
| Parameter binding | [parameter-binding.md](./parameter-binding.md) | Explicit [FromRoute], [FromBody], [FromQuery] |
| Results | [results.md](./results.md) | TypedResults for compile-time safety |
| Value objects | [value-objects.md](./value-objects.md) | TryParse for automatic binding |
| MIME versioning | [mime-versioning.md](./mime-versioning.md) | `application/vnd.{vendor}.{resource}.v{n}+json` |
| Route constraints | [route-constraints.md](./route-constraints.md) | Built-in and custom IRouteConstraint |
| OpenAPI | [openapi-configuration.md](./openapi-configuration.md) | Inline WithOpenApi(), Examples nested class |

## File Structure

```
Routes/
├── Loans/
│   ├── v1/
│   │   ├── LoanCreateRoute.cs
│   │   └── LoanGetRoute.cs
│   └── v2/
│       └── LoanCreateRoute.cs
└── Users/
    └── v1/
        └── UserGetRoute.cs
```

## Decision Rationale

- **Minimal APIs** -- lighter than controllers, direct HTTP-to-code mapping
- **MIME-type versioning** -- RESTful, clean URLs, industry standard (GitHub, Stripe)
- **Versioned namespaces** -- independent implementations per version, no shared code
- **DataAnnotations** -- built-in, source-generator friendly (.NET 10+), AOT compatible
- **Domain exceptions** -- clean handlers without try-catch; middleware handles conversion
- **Policy services** -- centralized auth, reusable, fail-fast pattern
- **Value objects** -- type safety; can't pass LibraryId where BookId is expected
