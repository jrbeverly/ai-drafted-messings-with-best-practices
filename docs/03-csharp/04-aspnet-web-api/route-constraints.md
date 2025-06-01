# Route Constraints

Validate route parameters. Type constraints. Custom constraints. Parameter validation at routing level.

## Principle

Validate route parameters before action execution. Ensure type safety. Prevent invalid routes from matching.

## Basic Type Constraints

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    // int constraint
    // Matches: /api/users/123
    // Doesn't match: /api/users/abc
    [HttpGet("{id:int}")]
    public IActionResult GetUser(int id)
    {
        var user = GetUserById(id);
        return Ok(user);
    }

    // guid constraint
    // Matches: /api/users/550e8400-e29b-41d4-a716-446655440000
    // Doesn't match: /api/users/not-a-guid
    [HttpGet("{id:guid}")]
    public IActionResult GetUserByGuid(Guid id)
    {
        var user = GetUserByGuid(id);
        return Ok(user);
    }

    // long constraint
    [HttpGet("{id:long}")]
    public IActionResult GetUserByLong(long id) => Ok();

    // decimal constraint
    [HttpGet("balance/{amount:decimal}")]
    public IActionResult GetByBalance(decimal amount) => Ok();

    // double constraint
    [HttpGet("rating/{score:double}")]
    public IActionResult GetByRating(double score) => Ok();

    // float constraint
    [HttpGet("price/{value:float}")]
    public IActionResult GetByPrice(float value) => Ok();

    // bool constraint
    [HttpGet("active/{isActive:bool}")]
    public IActionResult GetByActive(bool isActive) => Ok();
}
```

## String Constraints

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    // alpha - Letters only
    // Matches: /api/users/john
    // Doesn't match: /api/users/john123
    [HttpGet("{name:alpha}")]
    public IActionResult GetUserByName(string name) => Ok();

    // minlength - Minimum string length
    // Matches: /api/users/john
    // Doesn't match: /api/users/ab (too short)
    [HttpGet("{username:minlength(3)}")]
    public IActionResult GetByUsername(string username) => Ok();

    // maxlength - Maximum string length
    // Matches: /api/users/john
    // Doesn't match: /api/users/verylongusernamethatexceedslimit
    [HttpGet("{code:maxlength(10)}")]
    public IActionResult GetByCode(string code) => Ok();

    // length - Exact length or range
    // Matches: /api/users/12345 (exactly 5 chars)
    [HttpGet("{zipCode:length(5)}")]
    public IActionResult GetByZipCode(string zipCode) => Ok();

    // length range
    [HttpGet("{username:length(3,20)}")]
    public IActionResult GetByUsernameRange(string username) => Ok();
}
```

## Numeric Range Constraints

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    // min - Minimum value
    // Matches: /api/users/18 and above
    // Doesn't match: /api/users/17
    [HttpGet("age/{age:min(18)}")]
    public IActionResult GetByAge(int age) => Ok();

    // max - Maximum value
    [HttpGet("rating/{rating:max(5)}")]
    public IActionResult GetByRating(int rating) => Ok();

    // range - Min and max
    // Matches: /api/users/age/25 (18-100)
    [HttpGet("age/{age:range(18,100)}")]
    public IActionResult GetByAgeRange(int age) => Ok();
}
```

## Regular Expression Constraints

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    // regex - Regular expression pattern
    // Matches: /api/users/user-123
    // Pattern: user-{digits}
    [HttpGet("{id:regex(^user-\\d+$)}")]
    public IActionResult GetUserById(string id) => Ok();

    // Email pattern
    [HttpGet("by-email/{email:regex(^[\\w-\\.]+@[\\w-]+\\.[a-z]{{2,4}}$)}")]
    public IActionResult GetByEmail(string email) => Ok();

    // Phone number pattern (US format)
    [HttpGet("by-phone/{phone:regex(^\\d{{3}}-\\d{{3}}-\\d{{4}}$)}")]
    public IActionResult GetByPhone(string phone) => Ok();

    // ISO date format
    [HttpGet("by-date/{date:regex(^\\d{{4}}-\\d{{2}}-\\d{{2}}$)}")]
    public IActionResult GetByDate(string date) => Ok();
}
```

## Multiple Constraints

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    // Combine multiple constraints
    // int AND min AND max
    [HttpGet("{id:int:min(1):max(9999)}")]
    public IActionResult GetUser(int id) => Ok();

    // string AND minlength AND maxlength
    [HttpGet("{username:minlength(3):maxlength(20)}")]
    public IActionResult GetByUsername(string username) => Ok();

    // guid AND regex (custom format)
    [HttpGet("{id:guid:regex(^[0-9a-f]{{8}}-)}")]
    public IActionResult GetByGuid(Guid id) => Ok();
}
```

## Optional Parameters with Constraints

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    // Optional parameter with constraint
    // Matches: /api/users/filter
    // Matches: /api/users/filter/5
    [HttpGet("filter/{page:int?}")]
    public IActionResult FilterUsers(int page = 1)
    {
        var users = GetUsersPage(page);
        return Ok(users);
    }

    // Multiple optional parameters
    [HttpGet("search/{query?}/{page:int?}")]
    public IActionResult SearchUsers(string? query, int page = 1) => Ok();
}
```

## Custom Route Constraint

```csharp
// Custom constraint for API key format
public class ApiKeyConstraint : IRouteConstraint
{
    private static readonly Regex ApiKeyPattern = new(@"^[A-Z0-9]{32}$");

    public bool Match(
        HttpContext? httpContext,
        IRouter? route,
        string routeKey,
        RouteValueDictionary values,
        RouteDirection routeDirection)
    {
        if (values.TryGetValue(routeKey, out var value))
        {
            var apiKey = value?.ToString();
            return apiKey != null && ApiKeyPattern.IsMatch(apiKey);
        }

        return false;
    }
}

// Register
builder.Services.Configure<RouteOptions>(options =>
{
    options.ConstraintMap.Add("apikey", typeof(ApiKeyConstraint));
});

// Usage
[HttpGet("secure/{key:apikey}")]
public IActionResult SecureEndpoint(string key) => Ok();

// Matches: /api/secure/ABCD1234EFGH5678IJKL9012MNOP3456
// Doesn't match: /api/secure/invalid-key
```

## Custom Constraint with Dependencies

```csharp
// Constraint that checks if user exists
public class UserExistsConstraint : IRouteConstraint
{
    private readonly IServiceProvider _serviceProvider;

    public UserExistsConstraint(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public bool Match(
        HttpContext? httpContext,
        IRouter? route,
        string routeKey,
        RouteValueDictionary values,
        RouteDirection routeDirection)
    {
        if (values.TryGetValue(routeKey, out var value))
        {
            var userId = value?.ToString();

            if (string.IsNullOrEmpty(userId))
                return false;

            // Resolve service from DI
            using var scope = _serviceProvider.CreateScope();
            var userService = scope.ServiceProvider.GetRequiredService<IUserService>();

            // Check if user exists (synchronous check for routing)
            return userService.UserExistsAsync(userId).GetAwaiter().GetResult();
        }

        return false;
    }
}

// Register
builder.Services.Configure<RouteOptions>(options =>
{
    options.ConstraintMap.Add("userexists", typeof(UserExistsConstraint));
});

// Usage
[HttpGet("{userId:userexists}/profile")]
public IActionResult GetProfile(string userId) => Ok();

// Returns 404 automatically if user doesn't exist
```

## Date/DateTime Constraints

```csharp
// Custom constraint for date format
public class DateOnlyConstraint : IRouteConstraint
{
    public bool Match(
        HttpContext? httpContext,
        IRouter? route,
        string routeKey,
        RouteValueDictionary values,
        RouteDirection routeDirection)
    {
        if (values.TryGetValue(routeKey, out var value))
        {
            var dateStr = value?.ToString();
            return DateOnly.TryParse(dateStr, out _);
        }

        return false;
    }
}

// Register
builder.Services.Configure<RouteOptions>(options =>
{
    options.ConstraintMap.Add("dateonly", typeof(DateOnlyConstraint));
});

// Usage
[HttpGet("reports/{date:dateonly}")]
public IActionResult GetReport(DateOnly date) => Ok();

// Matches: /api/reports/2026-02-13
// Doesn't match: /api/reports/invalid-date
```

## Enum Constraints

```csharp
public enum OrderStatus
{
    Pending,
    Processing,
    Shipped,
    Delivered
}

// Custom enum constraint
public class EnumConstraint<TEnum> : IRouteConstraint where TEnum : struct, Enum
{
    public bool Match(
        HttpContext? httpContext,
        IRouter? route,
        string routeKey,
        RouteValueDictionary values,
        RouteDirection routeDirection)
    {
        if (values.TryGetValue(routeKey, out var value))
        {
            var enumStr = value?.ToString();
            return Enum.TryParse<TEnum>(enumStr, ignoreCase: true, out _);
        }

        return false;
    }
}

// Register
builder.Services.Configure<RouteOptions>(options =>
{
    options.ConstraintMap.Add("orderstatus", typeof(EnumConstraint<OrderStatus>));
});

// Usage
[HttpGet("orders/{status:orderstatus}")]
public IActionResult GetOrdersByStatus(OrderStatus status) => Ok();

// Matches: /api/orders/pending
// Matches: /api/orders/shipped
// Doesn't match: /api/orders/invalid
```

## Minimal API Constraints

```csharp
// Constraints in Minimal APIs
app.MapGet("/users/{id:int}", (int id) => Results.Ok($"User {id}"));

app.MapGet("/users/{id:guid}", (Guid id) => Results.Ok($"User {id}"));

app.MapGet("/users/{username:minlength(3)}", (string username) =>
    Results.Ok($"User {username}"));

app.MapGet("/users/age/{age:range(18,100)}", (int age) =>
    Results.Ok($"Age {age}"));

// Custom constraint
app.MapGet("/users/{id:userexists}", (string id) =>
    Results.Ok($"User {id} exists"));
```

## Catch-All Routes

```csharp
[ApiController]
[Route("api")]
public class FilesController : ControllerBase
{
    // Catch-all parameter (matches everything after files/)
    // Matches: /api/files/documents/2026/report.pdf
    [HttpGet("files/{**path}")]
    public IActionResult GetFile(string path)
    {
        // path = "documents/2026/report.pdf"
        var file = GetFileByPath(path);
        return File(file, "application/octet-stream");
    }

    // With constraint
    [HttpGet("static/{**path:regex(.*\\.(css|js|png|jpg)$)}")]
    public IActionResult GetStaticFile(string path) => Ok();
}
```

## Testing Route Constraints

```csharp
public class RouteConstraintTests
{
    [Theory]
    [InlineData("/api/users/123", HttpStatusCode.OK)] // Valid int
    [InlineData("/api/users/abc", HttpStatusCode.NotFound)] // Invalid int
    [InlineData("/api/users/0", HttpStatusCode.NotFound)] // Below min
    public async Task GetUser_IntConstraint_ValidatesCorrectly(
        string path,
        HttpStatusCode expectedStatus)
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        // Act
        var response = await client.GetAsync(path);

        // Assert
        Assert.Equal(expectedStatus, response.StatusCode);
    }

    [Theory]
    [InlineData("/api/users/john", HttpStatusCode.OK)] // Valid alpha
    [InlineData("/api/users/john123", HttpStatusCode.NotFound)] // Invalid alpha
    public async Task GetUser_AlphaConstraint_ValidatesCorrectly(
        string path,
        HttpStatusCode expectedStatus)
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        // Act
        var response = await client.GetAsync(path);

        // Assert
        Assert.Equal(expectedStatus, response.StatusCode);
    }
}
```

## Built-in Constraints Reference

```csharp
// Type constraints
"{id:int}"          // Integer
"{id:long}"         // Long
"{id:decimal}"      // Decimal
"{id:double}"       // Double
"{id:float}"        // Float
"{id:guid}"         // GUID
"{id:bool}"         // Boolean
"{date:datetime}"   // DateTime

// String constraints
"{name:alpha}"              // Letters only
"{name:minlength(3)}"       // Minimum length
"{name:maxlength(10)}"      // Maximum length
"{name:length(5)}"          // Exact length
"{name:length(3,10)}"       // Length range
"{name:regex(^[a-z]+$)}"    // Regular expression

// Numeric constraints
"{age:min(18)}"             // Minimum value
"{age:max(100)}"            // Maximum value
"{age:range(18,100)}"       // Value range

// Optional
"{id:int?}"                 // Optional parameter

// Catch-all
"{**path}"                  // Matches everything
```

## Guidelines

**When to Use Constraints:**
- Type safety (int, guid)
- Format validation (regex)
- Range validation (min, max, range)
- Prevent invalid route matching

**Performance:**
- Keep constraints simple
- Avoid expensive operations in constraints
- Cache results if possible
- Prefer built-in constraints

**Custom Constraints:**
- Implement IRouteConstraint
- Register in RouteOptions
- Test thoroughly
- Document behavior

**Best Practices:**
- Use most specific constraint possible
- Combine constraints when needed
- Test edge cases
- Return 404 for constraint failures

## Benefits

Type-safe. Catch errors at routing level.

Validation. Prevent invalid parameters.

Clear. Self-documenting routes.

Performance. Early rejection of bad requests.

## Related

- [attribute-routing.md](./attribute-routing.md) - Route configuration
- [model-binding.md](./model-binding.md) - Parameter binding
- [request-validation.md](./request-validation.md) - Input validation
- [custom-model-binders.md](./custom-model-binders.md) - Custom binding
