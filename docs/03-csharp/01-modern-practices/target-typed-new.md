# Target-Typed New

C# 9 target-typed new expressions omit type on right side when obvious from context.

## Principle

When the type is clear from context, omit redundant type specification on the right side of assignment.

## Basic Syntax

Omit type after new:

```csharp
// Traditional
User user = new User();
List<string> names = new List<string>();

// Target-typed new
User user = new();
List<string> names = new();
```

Type inferred from left side.

## Field Initialization

Use in field declarations:

```csharp
public class UserService
{
    private readonly List<string> _cache = new();
    private readonly Dictionary<string, User> _users = new();
    private readonly HttpClient _client = new();
}
```

## Constructor Injection

Cleaner DI field initialization:

```csharp
public class UserService
{
    private readonly IUserRepository _repository;
    private readonly ILogger<UserService> _logger;
    private readonly List<string> _processedIds = new();

    public UserService(IUserRepository repository, ILogger<UserService> logger)
    {
        _repository = repository;
        _logger = logger;
    }
}
```

## Method Return

Return statements:

```csharp
public User CreateUser(string name, string email)
{
    return new()
    {
        Id = Guid.NewGuid(),
        Name = name,
        Email = email,
        CreatedAt = DateTime.UtcNow
    };
}
```

## Collection Initialization

Initialize collections:

```csharp
List<User> users = new()
{
    new() { Name = "Alice", Email = "alice@example.com" },
    new() { Name = "Bob", Email = "bob@example.com" },
    new() { Name = "Charlie", Email = "charlie@example.com" }
};
```

## Method Arguments

Pass to method parameters:

```csharp
void ProcessUser(User user)
{
    // ...
}

// Call with target-typed new
ProcessUser(new()
{
    Name = "John",
    Email = "john@example.com"
});
```

## Conditional Expressions

Use in ternary operators:

```csharp
User? user = condition
    ? new() { Name = "John", Email = "john@example.com" }
    : null;
```

## Generic Types

Works with generic types:

```csharp
Dictionary<string, List<User>> usersByRole = new();

// Add to dictionary
usersByRole["Admin"] = new();
usersByRole["User"] = new();
```

## Real-World Examples

### API Response

```csharp
public async Task<IResult> CreateUserAsync(Request request, IUserService service)
{
    var user = await service.CreateUserAsync(request.Email, request.Name);

    return Results.Ok(new Response
    {
        UserId = user.Id,
        Email = user.Email,
        Name = user.Name
    });
    // Could also write: Results.Ok(new() { ... })
}
```

### Repository Pattern

```csharp
public class UserRepository : IUserRepository
{
    private readonly IAmazonDynamoDB _dynamoDb;
    private readonly Dictionary<string, User> _cache = new();

    public async Task<User?> GetByIdAsync(Guid id)
    {
        if (_cache.TryGetValue(id.ToString(), out var cached))
            return cached;

        var request = new GetItemRequest
        {
            TableName = "Users",
            Key = new()
            {
                ["PK"] = new() { S = $"USER#{id}" }
            }
        };

        var response = await _dynamoDb.GetItemAsync(request);
        return response.Item.Count > 0 ? MapToUser(response.Item) : null;
    }
}
```

### Service Layer

```csharp
public class LoanService : ILoanService
{
    private readonly ILoanRepository _repository;
    private readonly ILogger<LoanService> _logger;
    private readonly List<string> _recentLoans = new();

    public async Task<Loan> CreateLoanAsync(
        string libraryId,
        string userId,
        string bookId,
        int durationDays)
    {
        var loan = new Loan
        {
            Id = Guid.NewGuid().ToString(),
            LibraryId = libraryId,
            UserId = userId,
            BookId = bookId,
            DueDate = DateTime.UtcNow.AddDays(durationDays),
            Status = LoanStatus.Active
        };
        // Could write: var loan = new() { ... };

        await _repository.SaveAsync(loan);
        return loan;
    }
}
```

### Test Data

```csharp
[Fact]
public async Task CreateLoan_ValidRequest_ReturnsLoan()
{
    // Arrange
    var request = new Request
    {
        BookId = "book_123",
        DurationDays = 14
    };
    // Could write: var request = new() { ... };

    // Act
    var result = await _service.CreateLoanAsync("lib_456", "user_789", request);

    // Assert
    Assert.NotNull(result);
}
```

## When Not to Use

Ambiguous types:

```csharp
// UNCLEAR - what type?
var user = new();

// BETTER - explicit type
var user = new User();
```

Complex generics where type helps readability:

```csharp
// Less clear
var dict = new();

// More clear
Dictionary<string, List<User>> dict = new();
```

## Guidelines

**Use Target-Typed New:**
- Field initialization
- Return statements
- Method arguments
- When type is obvious from context

**Type Is Obvious:**
- Explicit type on left side
- Method return type is clear
- Parameter type is known

**Avoid When:**
- Using var (type not visible)
- Type is complex or ambiguous
- Readability suffers

**Consistency:**
- Use throughout codebase
- Don't mix with traditional syntax arbitrarily
- Prefer when it improves clarity

## Comparison

Traditional vs target-typed:

```csharp
// Traditional - repetitive
List<string> names = new List<string>();
Dictionary<string, User> users = new Dictionary<string, User>();
User user = new User { Name = "John" };

// Target-typed - concise
List<string> names = new();
Dictionary<string, User> users = new();
User user = new() { Name = "John" };
```

## Benefits

Less repetition. Type not specified twice.

Cleaner code. Removes redundant information.

Easier refactoring. Change type once on left side.

## Related

- [collection-expressions.md](./collection-expressions.md) - Modern collection syntax
- [init-only-properties.md](./init-only-properties.md) - Object initialization
- [primary-constructors.md](./primary-constructors.md) - Constructor patterns
