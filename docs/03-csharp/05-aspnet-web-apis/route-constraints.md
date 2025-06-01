# Route Constraints

Validate route parameters at routing time using built-in and custom constraints. Invalid format returns 404 (not routed), not 400.

## Why It Matters

- Fail fast before the handler executes
- Prevents routing to the wrong handler (e.g., GUID route won't match an integer)
- Keeps handler logic focused on business rules, not format checks

## Built-in Constraints

```csharp
"/{id:int}"               // integer
"/{id:guid}"              // GUID
"/{code:alpha}"           // letters only
"/{code:length(5)}"       // exact length
"/{code:minlength(2)}"    // min length
"/{age:range(18,120)}"    // numeric range
"/{id:min(1)}"            // minimum value
"/{sku:regex(^[A-Z]{{3}}-\\d{{4}}$)}" // regex pattern
"/{date:datetime}"        // valid date
```

Chain multiple: `"/{id:int:min(1)}"`, `"/{code:alpha:length(3)}"`

## Custom Constraint

```csharp
public class LibraryIdConstraint : IRouteConstraint
{
    public bool Match(HttpContext? ctx, IRouter? route, string routeKey,
        RouteValueDictionary values, RouteDirection direction)
    {
        var val = values[routeKey]?.ToString();
        return !string.IsNullOrEmpty(val) && val.StartsWith("lib_");
    }
}

// Register
builder.Services.Configure<RouteOptions>(o =>
    o.ConstraintMap.Add("libraryid", typeof(LibraryIdConstraint)));

// Use
group.MapGet("/{libraryId:libraryid}", ...)
```

## Constraint vs. Validation

| Route Constraint | DataAnnotations / IValidatableObject |
|------------------|--------------------------------------|
| Routing decision (404 if no match) | Business validation (400 if invalid) |
| Simple format checks | Complex rules, cross-property |
| Runs before handler | Runs after routing, before handler body |

Use constraints for **format**, validation for **business rules**.

## Pitfalls to Avoid

- Using constraints for complex business validation (use DataAnnotations)
- Database lookups inside a constraint
- Forgetting to register custom constraints in `RouteOptions`
- Over-constraining routes -- keep constraints simple

## Related

- [parameter-binding.md](./parameter-binding.md) -- parameter types
- [value-objects.md](./value-objects.md) -- TryParse as an alternative
- [minimal-api-routing.md](./minimal-api-routing.md) -- route configuration
