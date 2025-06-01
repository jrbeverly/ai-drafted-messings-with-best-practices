# Request Validation

Validate HTTP request data before processing. Use FluentValidation for complex rules. Return clear validation errors.

## Principle

Validate at the boundary. Fail fast with actionable errors. Use strongly-typed validators.

## Manual Validation

```csharp
app.MapPost("/users", async (
    CreateUserRequest request,
    IUserService userService) =>
{
    // Manual validation
    if (string.IsNullOrEmpty(request.Email))
    {
        return Results.ValidationProblem(new Dictionary<string, string[]>
        {
            ["Email"] = new[] { "Email is required" }
        });
    }

    if (!IsValidEmail(request.Email))
    {
        return Results.ValidationProblem(new Dictionary<string, string[]>
        {
            ["Email"] = new[] { "Email must be valid" }
        });
    }

    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});
```

## Data Annotations

```csharp
public record CreateUserRequest
{
    [Required(ErrorMessage = "Email is required")]
    [EmailAddress(ErrorMessage = "Email must be valid")]
    public string Email { get; init; } = string.Empty;

    [Required(ErrorMessage = "Name is required")]
    [StringLength(100, MinimumLength = 2, ErrorMessage = "Name must be between 2 and 100 characters")]
    public string Name { get; init; } = string.Empty;

    [Range(18, 120, ErrorMessage = "Age must be between 18 and 120")]
    public int Age { get; init; }
}

// Validation filter
app.MapPost("/users", async (
    CreateUserRequest request,
    IUserService userService) =>
{
    var validationResults = new List<ValidationResult>();
    var context = new ValidationContext(request);

    if (!Validator.TryValidateObject(request, context, validationResults, true))
    {
        var errors = validationResults
            .GroupBy(r => r.MemberNames.FirstOrDefault() ?? "")
            .ToDictionary(
                g => g.Key,
                g => g.Select(r => r.ErrorMessage ?? "").ToArray());

        return Results.ValidationProblem(errors);
    }

    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});
```

## FluentValidation

```csharp
// Install: dotnet add package FluentValidation

public record CreateUserRequest
{
    public string Email { get; init; } = string.Empty;
    public string Name { get; init; } = string.Empty;
    public int Age { get; init; }
}

public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Email must be valid")
            .MaximumLength(255).WithMessage("Email cannot exceed 255 characters");

        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .Length(2, 100).WithMessage("Name must be between 2 and 100 characters")
            .Matches(@"^[a-zA-Z\s]+$").WithMessage("Name can only contain letters and spaces");

        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120).WithMessage("Age must be between 18 and 120");
    }
}

// Register validators
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();

// Use in endpoint
app.MapPost("/users", async (
    CreateUserRequest request,
    IValidator<CreateUserRequest> validator,
    IUserService userService) =>
{
    var validationResult = await validator.ValidateAsync(request);

    if (!validationResult.IsValid)
    {
        return Results.ValidationProblem(
            validationResult.ToDictionary());
    }

    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});
```

## Validation Filter

```csharp
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var validator = context.HttpContext
            .RequestServices
            .GetService<IValidator<T>>();

        if (validator is null)
        {
            return await next(context);
        }

        var argument = context.Arguments
            .OfType<T>()
            .FirstOrDefault();

        if (argument is null)
        {
            return await next(context);
        }

        var validationResult = await validator.ValidateAsync(argument);

        if (!validationResult.IsValid)
        {
            return Results.ValidationProblem(
                validationResult.ToDictionary());
        }

        return await next(context);
    }
}

// Apply to endpoint
app.MapPost("/users", CreateUser)
    .AddEndpointFilter<ValidationFilter<CreateUserRequest>>();

// Or create extension method
public static class ValidationExtensions
{
    public static RouteHandlerBuilder WithValidation<T>(
        this RouteHandlerBuilder builder) where T : class
    {
        return builder.AddEndpointFilter<ValidationFilter<T>>();
    }
}

// Usage
app.MapPost("/users", CreateUser)
    .WithValidation<CreateUserRequest>();
```

## Complex Validation Rules

```csharp
public class CreateLoanRequestValidator : AbstractValidator<CreateLoanRequest>
{
    public CreateLoanRequestValidator()
    {
        RuleFor(x => x.BookId)
            .NotEmpty().WithMessage("BookId is required");

        RuleFor(x => x.UserId)
            .NotEmpty().WithMessage("UserId is required");

        RuleFor(x => x.DueDate)
            .GreaterThan(DateTime.UtcNow).WithMessage("Due date must be in the future")
            .LessThan(DateTime.UtcNow.AddMonths(3)).WithMessage("Due date cannot be more than 3 months from now");

        // Conditional validation
        When(x => x.IsRenewal, () =>
        {
            RuleFor(x => x.PreviousLoanId)
                .NotEmpty().WithMessage("PreviousLoanId is required for renewals");
        });

        // Custom validation
        RuleFor(x => x.Email)
            .Must(BeUniqueEmail).WithMessage("Email is already in use");
    }

    private bool BeUniqueEmail(string email)
    {
        // Check database for uniqueness
        return true; // Implementation
    }
}
```

## Async Validation

```csharp
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    private readonly IUserRepository _userRepository;

    public CreateUserRequestValidator(IUserRepository userRepository)
    {
        _userRepository = userRepository;

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MustAsync(BeUniqueEmail).WithMessage("Email is already in use");
    }

    private async Task<bool> BeUniqueEmail(string email, CancellationToken cancellationToken)
    {
        var existingUser = await _userRepository.GetByEmailAsync(email);
        return existingUser is null;
    }
}
```

## Dependent Properties

```csharp
public class CreateEventRequestValidator : AbstractValidator<CreateEventRequest>
{
    public CreateEventRequestValidator()
    {
        RuleFor(x => x.StartDate)
            .NotEmpty()
            .GreaterThan(DateTime.UtcNow);

        RuleFor(x => x.EndDate)
            .NotEmpty()
            .GreaterThan(x => x.StartDate)
            .WithMessage("End date must be after start date");
    }
}
```

## Collection Validation

```csharp
public class CreateOrderRequestValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderRequestValidator()
    {
        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("Order must contain at least one item")
            .Must(x => x.Count <= 100).WithMessage("Order cannot contain more than 100 items");

        RuleForEach(x => x.Items)
            .SetValidator(new OrderItemValidator());
    }
}

public class OrderItemValidator : AbstractValidator<OrderItem>
{
    public OrderItemValidator()
    {
        RuleFor(x => x.ProductId)
            .NotEmpty().WithMessage("ProductId is required");

        RuleFor(x => x.Quantity)
            .GreaterThan(0).WithMessage("Quantity must be greater than 0")
            .LessThanOrEqualTo(999).WithMessage("Quantity cannot exceed 999");
    }
}
```

## Custom Error Messages

```csharp
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Please provide an email address")
            .EmailAddress().WithMessage("'{PropertyValue}' is not a valid email address")
            .MaximumLength(255).WithMessage("Email cannot be longer than {MaxLength} characters");

        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120)
            .WithMessage("Age must be between {From} and {To}, but was {PropertyValue}");
    }
}
```

## Error Response Format

```csharp
// Default ValidationProblem response
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Email": [
      "Email is required",
      "Email must be valid"
    ],
    "Name": [
      "Name must be between 2 and 100 characters"
    ]
  }
}

// Custom error response
app.MapPost("/users", async (
    CreateUserRequest request,
    IValidator<CreateUserRequest> validator,
    IUserService userService) =>
{
    var validationResult = await validator.ValidateAsync(request);

    if (!validationResult.IsValid)
    {
        var errors = validationResult.Errors
            .GroupBy(e => e.PropertyName)
            .ToDictionary(
                g => g.Key,
                g => g.Select(e => new
                {
                    message = e.ErrorMessage,
                    attemptedValue = e.AttemptedValue,
                    severity = e.Severity.ToString()
                }).ToArray());

        return Results.BadRequest(new
        {
            message = "Validation failed",
            errors
        });
    }

    var user = await userService.CreateUserAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});
```

## Guidelines

**Validation Strategy:**
- Validate at API boundary (endpoint level)
- Use FluentValidation for complex rules
- Return all errors at once (not just first error)
- Include field names and clear messages

**Error Messages:**
- User-friendly, actionable messages
- Include expected format/range
- Avoid technical jargon
- Consistent tone

**Performance:**
- Avoid expensive validation in synchronous code
- Cache validation results when appropriate
- Use MustAsync for database checks
- Consider validation impact on latency

**Testing:**
- Test validators independently
- Test edge cases (null, empty, boundary values)
- Verify error messages
- Test async validation logic

## Benefits

Safe. Invalid data rejected early.

Clear. Actionable error messages.

Consistent. Centralized validation rules.

Maintainable. Easy to add/modify rules.

## Related

- [minimal-api-basics.md](./minimal-api-basics.md) - Endpoint definition
- [endpoint-filters.md](./endpoint-filters.md) - Validation filters
- [api-error-handling.md](../../08-api-design/api-error-handling.md) - Error responses
- [problem-details.md](./problem-details.md) - RFC 7807 format
