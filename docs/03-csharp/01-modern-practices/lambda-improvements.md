# Lambda Improvements

Modern C# lambda enhancements: natural types, attributes, explicit return types, and static lambdas (C# 9-12).

## Why It Matters

- Natural types enable `var` assignment and direct method-group passing
- Attribute support makes lambdas first-class for Minimal API parameter binding
- Static lambdas prevent accidental closures in performance-critical code

## Key Recommendations

**Natural types** (C# 10) -- assign lambdas to `var`:
```csharp
var parse = (string s) => int.Parse(s);
var multiply = (int x, int y) => x * y;
```

**Attributes on parameters** (C# 10) -- essential for Minimal APIs:
```csharp
app.MapGet("/users/{id}", async ([FromRoute] string id, [FromServices] IUserService svc) =>
    await svc.GetUserAsync(id) is { } user ? Results.Ok(user) : Results.NotFound());
```

**Explicit return types** when disambiguation is needed:
```csharp
var compute = double (int x, int y) => (double)x / y;
```

**Static lambdas** (C# 9) -- prevent captures, avoid closure allocation:
```csharp
var multiply = static (int x) => x * 10;
// static (int x) => x * factor; // Compiler error: cannot capture
```

**Expression body** for simple cases, **statement body** for complex logic or async.

**Discard parameters** with `_`:
```csharp
Func<int, int, int> addTen = (x, _) => x + 10;
```

## Pitfalls to Avoid

- Closures over local variables allocate (use `static` on hot paths to prevent)
- Generic lambdas (`<T>(T val) => val`) are proposed but not yet available
- Async lambdas always require statement body with `return`
