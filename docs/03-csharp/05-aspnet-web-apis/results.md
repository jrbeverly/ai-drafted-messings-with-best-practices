# Results

Typed return values for minimal API endpoints using `Results` and `TypedResults`.

## Why It Matters

- `TypedResults` gives compile-time safety -- the compiler verifies all return paths
- `Results<T1, T2, ...>` union types document the API contract and auto-generate OpenAPI responses
- Consistent status code usage improves API predictability

## Common Results

```csharp
Results.Ok(response)                          // 200
Results.Created($"/api/books/{id}", response) // 201
Results.NoContent()                           // 204
Results.BadRequest("Invalid input")           // 400
Results.Unauthorized()                        // 401
Results.Forbid()                              // 403
Results.NotFound()                            // 404
Results.Conflict()                            // 409
Results.Problem("An error occurred")          // 500 (Problem Details)
Results.ValidationProblem(errors)             // 400 (ValidationProblemDetails)
```

## TypedResults with Union Types

```csharp
public static async Task<Results<Ok<Response>, NotFound>> HandleAsync(...)
{
    var book = await repo.GetAsync(id);
    return book != null
        ? TypedResults.Ok(new Response { ... })
        : TypedResults.NotFound();
}
```

OpenAPI infers all possible responses from the return type.

## Status Code Selection

| Verb | Success | Meaning |
|------|---------|---------|
| GET, PUT, PATCH | 200 | Successful with body |
| POST | 201 | Resource created |
| DELETE, PUT (no body) | 204 | No content |

Client errors: 400 (validation), 401 (unauthn), 403 (unauthz), 404 (missing), 409 (conflict).

## OpenAPI Integration

Explicit:
```csharp
.Produces<Response>(201).Produces(400).Produces(401)
```

Or implicit via union return type -- both generate the same spec.

## Key Recommendations

- Prefer `TypedResults` over `Results` for compile-time safety
- Declare `Results<>` union type listing all possible outcomes
- Use `Created()` with a location header for POST endpoints
- Return Problem Details for all 4xx/5xx errors

## Pitfalls to Avoid

- Returning `IResult` without `Produces()` (OpenAPI won't document responses)
- Inconsistent status codes for the same operation type across endpoints
- Missing location header on 201 Created responses
- Returning body on 204 No Content

## Related

- [api-error-responses.md](./api-error-responses.md) -- Problem Details
- [request-response-pattern.md](./request-response-pattern.md) -- response DTOs
