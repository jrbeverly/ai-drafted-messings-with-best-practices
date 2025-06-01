# OpenAPI Configuration

Rich inline API documentation configured directly in route registration using `WithOpenApi()` and an `Examples` nested class.

## Why It Matters

- Docs stay in sync with code -- changes to endpoints automatically update the spec
- Rich metadata improves developer experience for API consumers
- Swagger UI provides interactive exploration in development

## Basic Setup

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Library API", Version = "v1"
    });
});

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

## Inline Metadata

```csharp
group.MapPost("/create", Handler.HandleAsync)
    .WithName("CreateLoan")
    .WithTags("Loans")
    .WithOpenApi(op =>
    {
        op.Summary = "Create book loan";
        op.Description = "Creates a new loan for a patron.";
        return op;
    });
```

## Examples Nested Class

```csharp
public static class Examples
{
    public const string RequestExample = """
        { "bookId": "book_ABC123", "durationDays": 14 }
        """;
    public const string ResponseExample = """
        { "loanId": "loan_XYZ789", "dueDate": "2026-01-27T23:59:59Z" }
        """;
}
```

Wire into `WithOpenApi()` by setting `operation.RequestBody.Content["application/json"].Example`.

## Documenting Responses

```csharp
.Produces<Response>(201, "application/vnd.library.loan-create.v1+json")
.Produces(StatusCodes.Status400BadRequest)
.Produces(StatusCodes.Status401Unauthorized)
```

Or use `TypedResults` with `Results<Created<Response>, BadRequest, NotFound>` for automatic inference.

## Versioned Media Types

```csharp
.AcceptsVersioned<Request>(ResourceName, Version)
.ProducesVersioned<Response>(ResourceName, Version)
```

## Deprecation

```csharp
.WithOpenApi(op => { op.Deprecated = true; return op; })
```

## Key Recommendations

- Use `WithOpenApi()` on every endpoint with at least summary and description
- Include realistic request/response examples via raw string literals
- Document all possible status codes with `Produces()`
- Group endpoints with `.WithTags()` at the group level

## Pitfalls to Avoid

- Skipping OpenAPI metadata (consumers rely on generated docs)
- Examples that don't match actual request/response shape
- Forgetting error response status codes in `Produces()`

## Related

- [endpoint-organization.md](./endpoint-organization.md) -- Examples nested class
- [mime-versioning.md](./mime-versioning.md) -- versioned media types
- [results.md](./results.md) -- Produces and TypedResults
