# CQRS Pattern

Command Query Responsibility Segregation. Separate reads from writes for clarity and scalability.

## Principle

Commands change state. Queries return data. Never mix the two. Different models for reading and writing.

## Basic Pattern

Commands (writes):

```csharp
// Commands change state, return void or ID
public record CreateUserCommand(string Email, string Name);

public class CreateUserHandler
{
    private readonly IUserRepository _repository;

    public async Task<string> HandleAsync(CreateUserCommand command)
    {
        var user = new User
        {
            Id = Guid.NewGuid().ToString(),
            Email = command.Email,
            Name = command.Name
        };

        await _repository.SaveAsync(user);

        return user.Id; // Return only ID, not full object
    }
}
```

Queries (reads):

```csharp
// Queries return data, don't change state
public record GetUserQuery(string UserId);

public record UserDto(string Id, string Email, string Name);

public class GetUserHandler
{
    private readonly IUserRepository _repository;

    public async Task<UserDto> HandleAsync(GetUserQuery query)
    {
        var user = await _repository.GetByIdAsync(query.UserId);

        if (user == null)
            throw new NotFoundException("User", query.UserId);

        return new UserDto(user.Id, user.Email, user.Name);
    }
}
```

## Command Structure

Commands are requests to change state:

```csharp
// Command - imperative, action-oriented
public record UpdateUserCommand(
    string UserId,
    string? Name,
    string? Email
);

public record DeleteLoanCommand(string LoanId);

public record ProcessOverdueLoansCommand(DateTime AsOfDate);
```

Command handler:

```csharp
public class UpdateUserHandler
{
    private readonly IUserRepository _repository;
    private readonly IUserPolicy _policy;

    public async Task HandleAsync(UpdateUserCommand command, string requesterId)
    {
        // Validate permissions
        await _policy.RequireCanUpdateUserAsync(requesterId, command.UserId);

        // Get entity
        var user = await _repository.GetByIdAsync(command.UserId)
            ?? throw new NotFoundException("User", command.UserId);

        // Update
        if (command.Name != null) user.Name = command.Name;
        if (command.Email != null) user.Email = command.Email;

        // Persist
        await _repository.SaveAsync(user);

        // No return value (or just success/failure)
    }
}
```

## Query Structure

Queries return data without side effects:

```csharp
// Query - request for data
public record GetUserQuery(string UserId);

public record SearchUsersQuery(
    string? SearchTerm,
    int Page,
    int Limit
);

public record GetUserLoansQuery(
    string UserId,
    LoanStatus? Status
);
```

Query handler:

```csharp
public class SearchUsersHandler
{
    private readonly IUserReadRepository _repository;

    public async Task<PagedResult<UserDto>> HandleAsync(SearchUsersQuery query)
    {
        // Read-only, no side effects
        var users = await _repository.SearchAsync(
            query.SearchTerm,
            query.Page,
            query.Limit
        );

        return new PagedResult<UserDto>
        {
            Items = users.Select(u => new UserDto(u.Id, u.Email, u.Name)).ToList(),
            Total = await _repository.CountAsync(query.SearchTerm),
            Page = query.Page,
            Limit = query.Limit
        };
    }
}
```

## Read and Write Models

Different models for reading and writing:

```csharp
// Write model - domain entity with behavior
public class User
{
    public required string Id { get; init; }
    public string Email { get; private set; } = string.Empty;
    public string Name { get; private set; } = string.Empty;
    public DateTime CreatedAt { get; init; }
    public DateTime? LastLoginAt { get; private set; }

    public void UpdateEmail(string email)
    {
        if (string.IsNullOrEmpty(email) || !email.Contains('@'))
            throw new ArgumentException("Invalid email");

        Email = email;
    }

    public void RecordLogin()
    {
        LastLoginAt = DateTime.UtcNow;
    }
}

// Read model - flat DTO optimized for display
public record UserListDto(
    string Id,
    string Email,
    string Name,
    int ActiveLoans,
    DateTime? LastLoginAt
);
```

## Separate Repositories

Different repositories for reads and writes:

```csharp
// Write repository - optimized for consistency
public interface IUserRepository
{
    Task<User?> GetByIdAsync(string id);
    Task SaveAsync(User user);
    Task DeleteAsync(string id);
}

// Read repository - optimized for queries
public interface IUserReadRepository
{
    Task<UserListDto[]> SearchAsync(string? searchTerm, int page, int limit);
    Task<int> CountAsync(string? searchTerm);
    Task<UserDetailDto?> GetDetailAsync(string id);
    Task<UserListDto[]> GetByIdsAsync(string[] ids);
}
```

Implementation can use same storage or different:

```csharp
// Same DynamoDB table, different query patterns
public class DynamoDbUserReadRepository : IUserReadRepository
{
    private readonly IAmazonDynamoDB _dynamoDb;

    public async Task<UserListDto[]> SearchAsync(
        string? searchTerm,
        int page,
        int limit)
    {
        // Query optimized for read performance
        // May denormalize data, use indexes, etc.
    }
}
```

## Minimal API Integration

Commands and queries in endpoints:

```csharp
// Command endpoint
app.MapPost("/api/users", async (
    [FromBody] CreateUserRequest request,
    CreateUserHandler handler) =>
{
    var command = new CreateUserCommand(request.Email, request.Name);
    var userId = await handler.HandleAsync(command);

    return Results.Created($"/api/users/{userId}", new { Id = userId });
});

// Query endpoint
app.MapGet("/api/users/{id}", async (
    string id,
    GetUserHandler handler) =>
{
    var query = new GetUserQuery(id);
    var user = await handler.HandleAsync(query);

    return Results.Ok(user);
});

// Query endpoint with parameters
app.MapGet("/api/users", async (
    [FromQuery] string? search,
    [FromQuery] int page,
    [FromQuery] int limit,
    SearchUsersHandler handler) =>
{
    var query = new SearchUsersQuery(search, page, limit);
    var result = await handler.HandleAsync(query);

    return Results.Ok(result);
});
```

## Validation

Validate commands before handling:

```csharp
public record CreateUserCommand(string Email, string Name) : IValidatableObject
{
    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (string.IsNullOrWhiteSpace(Email))
            yield return new ValidationResult("Email is required", [nameof(Email)]);

        if (!Email.Contains('@'))
            yield return new ValidationResult("Email must be valid", [nameof(Email)]);

        if (string.IsNullOrWhiteSpace(Name))
            yield return new ValidationResult("Name is required", [nameof(Name)]);
    }
}
```

## Event Sourcing Variant

Commands produce events (advanced):

```csharp
public record UserCreatedEvent(
    string UserId,
    string Email,
    string Name,
    DateTime CreatedAt
);

public class CreateUserHandler
{
    private readonly IEventStore _eventStore;

    public async Task<string> HandleAsync(CreateUserCommand command)
    {
        var userId = Guid.NewGuid().ToString();

        var @event = new UserCreatedEvent(
            userId,
            command.Email,
            command.Name,
            DateTime.UtcNow
        );

        await _eventStore.AppendAsync(userId, @event);

        return userId;
    }
}
```

## Mediator Pattern

Use MediatR for routing:

```bash
dotnet add package MediatR
```

```csharp
public record CreateUserCommand(string Email, string Name) : IRequest<string>;

public class CreateUserHandler : IRequestHandler<CreateUserCommand, string>
{
    private readonly IUserRepository _repository;

    public async Task<string> Handle(
        CreateUserCommand command,
        CancellationToken cancellationToken)
    {
        var user = new User
        {
            Id = Guid.NewGuid().ToString(),
            Email = command.Email,
            Name = command.Name
        };

        await _repository.SaveAsync(user);

        return user.Id;
    }
}

// Usage
var userId = await _mediator.Send(new CreateUserCommand("john@example.com", "John"));
```

## Folder Structure

Organize by feature with CQRS:

```
LibraryService.Application/
├── Users/
│   ├── Commands/
│   │   ├── CreateUser/
│   │   │   ├── CreateUserCommand.cs
│   │   │   └── CreateUserHandler.cs
│   │   └── UpdateUser/
│   │       ├── UpdateUserCommand.cs
│   │       └── UpdateUserHandler.cs
│   └── Queries/
│       ├── GetUser/
│       │   ├── GetUserQuery.cs
│       │   ├── GetUserHandler.cs
│       │   └── UserDto.cs
│       └── SearchUsers/
│           ├── SearchUsersQuery.cs
│           ├── SearchUsersHandler.cs
│           └── UserListDto.cs
└── Loans/
    ├── Commands/
    │   └── CreateLoan/
    └── Queries/
        └── GetUserLoans/
```

## Testing

Test commands and queries separately:

```csharp
// Test command
[Fact]
public async Task HandleAsync_ValidCommand_CreatesUser()
{
    var repository = Substitute.For<IUserRepository>();
    var handler = new CreateUserHandler(repository);

    var command = new CreateUserCommand("john@example.com", "John Doe");

    var userId = await handler.HandleAsync(command);

    Assert.NotEmpty(userId);
    await repository.Received(1).SaveAsync(Arg.Is<User>(u =>
        u.Email == "john@example.com" &&
        u.Name == "John Doe"
    ));
}

// Test query
[Fact]
public async Task HandleAsync_ExistingUser_ReturnsDto()
{
    var repository = Substitute.For<IUserReadRepository>();
    var user = new UserDto("user_123", "john@example.com", "John Doe");
    repository.GetByIdAsync("user_123").Returns(user);

    var handler = new GetUserHandler(repository);
    var query = new GetUserQuery("user_123");

    var result = await handler.HandleAsync(query);

    Assert.Equal("john@example.com", result.Email);
}
```

## Guidelines

**Commands:**
- Change state
- Return void or ID only
- Validate input
- Enforce business rules
- Can fail with exceptions

**Queries:**
- Read-only
- No side effects
- Return DTOs, not domain entities
- Optimize for performance
- Never throw NotFoundException (return null)

**Separation:**
- Separate handlers
- Separate repositories
- Different models
- Different optimization strategies

**When to Use:**
- Complex domain logic
- Different read/write patterns
- High read-to-write ratio
- Need for read optimization

## Benefits

Clarity. Clear separation of concerns.

Scalability. Optimize reads and writes independently.

Flexibility. Use different storage for reads and writes.

Performance. Tailored models for each operation.

## Related

- [clean-architecture.md](./clean-architecture.md) - Layered architecture
- [domain-driven-design.md](./domain-driven-design.md) - Domain modeling
- [event-sourcing.md](./event-sourcing.md) - Event-based persistence
