# Clean Architecture

Organize code by layers with dependency inversion. Business logic independent of frameworks and infrastructure.

## Principle

Dependencies point inward. Core domain has no dependencies. Infrastructure and presentation depend on domain, not vice versa.

## Layer Structure

Four concentric layers:

```
┌─────────────────────────────────────┐
│         Presentation Layer          │  Controllers, APIs, UI
│  ┌───────────────────────────────┐  │
│  │     Application Layer         │  │  Use cases, orchestration
│  │  ┌─────────────────────────┐  │  │
│  │  │    Domain Layer         │  │  │  Entities, business logic
│  │  │  ┌───────────────────┐  │  │  │
│  │  │  │  Infrastructure   │  │  │  │  Database, external APIs
│  │  │  └───────────────────┘  │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

Dependencies flow: Presentation → Application → Domain ← Infrastructure

## Project Structure

Organize by layer:

```
LibraryService/
├── LibraryService.Domain/           # Core business logic
│   ├── Entities/
│   │   ├── User.cs
│   │   ├── Book.cs
│   │   └── Loan.cs
│   ├── Interfaces/
│   │   ├── IUserRepository.cs
│   │   └── ILoanRepository.cs
│   ├── Exceptions/
│   │   ├── DomainException.cs
│   │   └── NotFoundException.cs
│   └── ValueObjects/
│       └── Email.cs
├── LibraryService.Application/      # Use cases
│   ├── Users/
│   │   ├── CreateUser/
│   │   │   ├── CreateUserCommand.cs
│   │   │   └── CreateUserHandler.cs
│   │   └── GetUser/
│   │       ├── GetUserQuery.cs
│   │       └── GetUserHandler.cs
│   └── Loans/
│       └── CreateLoan/
│           ├── CreateLoanCommand.cs
│           └── CreateLoanHandler.cs
├── LibraryService.Infrastructure/   # Implementation details
│   ├── Repositories/
│   │   ├── UserRepository.cs
│   │   └── LoanRepository.cs
│   └── ExternalServices/
│       └── EmailService.cs
└── LibraryService.Api/              # HTTP layer
    ├── Controllers/ or Routes/
    └── Program.cs
```

## Domain Layer

Pure business logic, no dependencies:

```csharp
// LibraryService.Domain/Entities/Loan.cs
namespace LibraryService.Domain.Entities;

public class Loan
{
    public required string Id { get; init; }
    public required string UserId { get; init; }
    public required string BookId { get; init; }
    public required DateTime DueDate { get; init; }
    public LoanStatus Status { get; private set; }

    // Business logic
    public void Return()
    {
        if (Status == LoanStatus.Returned)
            throw new InvalidOperationException("Loan already returned");

        Status = LoanStatus.Returned;
    }

    public bool IsOverdue()
    {
        return Status == LoanStatus.Active && DateTime.UtcNow > DueDate;
    }

    public void MarkOverdue()
    {
        if (IsOverdue())
            Status = LoanStatus.Overdue;
    }
}

public enum LoanStatus
{
    Active,
    Returned,
    Overdue
}
```

Domain interfaces (no implementation):

```csharp
// LibraryService.Domain/Interfaces/ILoanRepository.cs
namespace LibraryService.Domain.Interfaces;

public interface ILoanRepository
{
    Task<Loan?> GetByIdAsync(string id);
    Task<Loan[]> GetByUserIdAsync(string userId);
    Task SaveAsync(Loan loan);
    Task DeleteAsync(string id);
}
```

## Application Layer

Use cases and orchestration:

```csharp
// LibraryService.Application/Loans/CreateLoan/CreateLoanCommand.cs
namespace LibraryService.Application.Loans.CreateLoan;

public record CreateLoanCommand(
    string UserId,
    string BookId,
    int DurationDays
);

public record CreateLoanResult(
    string LoanId,
    DateTime DueDate
);
```

Handler with business logic:

```csharp
// LibraryService.Application/Loans/CreateLoan/CreateLoanHandler.cs
namespace LibraryService.Application.Loans.CreateLoan;

public class CreateLoanHandler
{
    private readonly ILoanRepository _loanRepository;
    private readonly IBookRepository _bookRepository;
    private readonly IEmailService _emailService;

    public CreateLoanHandler(
        ILoanRepository loanRepository,
        IBookRepository bookRepository,
        IEmailService emailService)
    {
        _loanRepository = loanRepository;
        _bookRepository = bookRepository;
        _emailService = emailService;
    }

    public async Task<CreateLoanResult> HandleAsync(CreateLoanCommand command)
    {
        // Validate business rules
        var book = await _bookRepository.GetByIdAsync(command.BookId)
            ?? throw new NotFoundException("Book", command.BookId);

        if (!book.IsAvailable)
            throw new BookUnavailableException(command.BookId);

        // Create domain entity
        var loan = new Loan
        {
            Id = Guid.NewGuid().ToString(),
            UserId = command.UserId,
            BookId = command.BookId,
            DueDate = DateTime.UtcNow.AddDays(command.DurationDays),
            Status = LoanStatus.Active
        };

        // Persist
        await _loanRepository.SaveAsync(loan);

        // Side effects
        await _emailService.SendLoanConfirmationAsync(command.UserId, loan);

        return new CreateLoanResult(loan.Id, loan.DueDate);
    }
}
```

## Infrastructure Layer

Concrete implementations:

```csharp
// LibraryService.Infrastructure/Repositories/LoanRepository.cs
namespace LibraryService.Infrastructure.Repositories;

public class LoanRepository : ILoanRepository
{
    private readonly IAmazonDynamoDB _dynamoDb;
    private readonly string _tableName;

    public LoanRepository(IAmazonDynamoDB dynamoDb, IOptions<DatabaseOptions> options)
    {
        _dynamoDb = dynamoDb;
        _tableName = options.Value.TableName;
    }

    public async Task<Loan?> GetByIdAsync(string id)
    {
        var request = new GetItemRequest
        {
            TableName = _tableName,
            Key = new Dictionary<string, AttributeValue>
            {
                ["PK"] = new AttributeValue { S = $"LOAN#{id}" }
            }
        };

        var response = await _dynamoDb.GetItemAsync(request);

        return response.Item.Count > 0
            ? MapToLoan(response.Item)
            : null;
    }

    public async Task SaveAsync(Loan loan)
    {
        var request = new PutItemRequest
        {
            TableName = _tableName,
            Item = MapToItem(loan)
        };

        await _dynamoDb.PutItemAsync(request);
    }

    private Loan MapToLoan(Dictionary<string, AttributeValue> item)
    {
        // Mapping logic
    }

    private Dictionary<string, AttributeValue> MapToItem(Loan loan)
    {
        // Mapping logic
    }
}
```

## Presentation Layer

HTTP endpoints (minimal API):

```csharp
// LibraryService.Api/Routes/Loans/LoanCreateRoute.cs
namespace LibraryService.Routes.Loans.v1;

public static class LoanCreateRoute
{
    public record Request
    {
        [Required]
        public string BookId { get; init; } = string.Empty;

        [Range(1, 90)]
        public int DurationDays { get; init; } = 14;
    }

    public static class Handler
    {
        public static async Task<IResult> HandleAsync(
            [FromRoute] string userId,
            [FromBody] Request request,
            CreateLoanHandler handler)
        {
            var command = new CreateLoanCommand(
                userId,
                request.BookId,
                request.DurationDays
            );

            var result = await handler.HandleAsync(command);

            return Results.Ok(new
            {
                LoanId = result.LoanId,
                DueDate = result.DueDate
            });
        }
    }
}
```

## Dependency Registration

Wire up layers in Program.cs:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Infrastructure (depends on Domain interfaces)
builder.Services.AddScoped<ILoanRepository, LoanRepository>();
builder.Services.AddScoped<IBookRepository, BookRepository>();
builder.Services.AddScoped<IEmailService, EmailService>();

// Application (depends on Domain interfaces)
builder.Services.AddScoped<CreateLoanHandler>();
builder.Services.AddScoped<GetLoanHandler>();

// AWS
builder.Services.AddSingleton<IAmazonDynamoDB, AmazonDynamoDBClient>();

var app = builder.Build();
```

## Testing

Test layers independently:

```csharp
// Unit test domain logic (no dependencies)
[Fact]
public void Return_ActiveLoan_SetsStatusToReturned()
{
    var loan = new Loan
    {
        Id = "loan_123",
        Status = LoanStatus.Active,
        // ...
    };

    loan.Return();

    Assert.Equal(LoanStatus.Returned, loan.Status);
}

// Test application handler (mock infrastructure)
[Fact]
public async Task HandleAsync_AvailableBook_CreatesLoan()
{
    var loanRepo = Substitute.For<ILoanRepository>();
    var bookRepo = Substitute.For<IBookRepository>();
    var emailService = Substitute.For<IEmailService>();

    bookRepo.GetByIdAsync("book_123")
        .Returns(new Book { Id = "book_123", IsAvailable = true });

    var handler = new CreateLoanHandler(loanRepo, bookRepo, emailService);

    var command = new CreateLoanCommand("user_123", "book_123", 14);
    var result = await handler.HandleAsync(command);

    Assert.NotNull(result.LoanId);
    await loanRepo.Received(1).SaveAsync(Arg.Any<Loan>());
}
```

## Dependency Rules

Enforce with project references:

```xml
<!-- LibraryService.Domain.csproj - No dependencies -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
</Project>

<!-- LibraryService.Application.csproj - Depends on Domain only -->
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <ProjectReference Include="..\LibraryService.Domain\LibraryService.Domain.csproj" />
  </ItemGroup>
</Project>

<!-- LibraryService.Infrastructure.csproj - Depends on Domain -->
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <ProjectReference Include="..\LibraryService.Domain\LibraryService.Domain.csproj" />
  </ItemGroup>
</Project>

<!-- LibraryService.Api.csproj - Depends on all layers -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <ItemGroup>
    <ProjectReference Include="..\LibraryService.Domain\LibraryService.Domain.csproj" />
    <ProjectReference Include="..\LibraryService.Application\LibraryService.Application.csproj" />
    <ProjectReference Include="..\LibraryService.Infrastructure\LibraryService.Infrastructure.csproj" />
  </ItemGroup>
</Project>
```

## Guidelines

**Domain Layer:**
- No dependencies on other projects
- Pure business logic
- Define interfaces, don't implement infrastructure
- Rich domain models with behavior

**Application Layer:**
- Orchestrate use cases
- Depend only on Domain
- No infrastructure concerns
- DTOs for input/output

**Infrastructure Layer:**
- Implement Domain interfaces
- Database, external APIs, file system
- Framework-specific code
- Don't leak implementation details

**Presentation Layer:**
- Thin HTTP/UI layer
- Map to/from Application DTOs
- Handle authentication/authorization
- No business logic

## Benefits

Testability. Test business logic without infrastructure.

Independence. Core logic independent of frameworks.

Flexibility. Swap implementations easily.

Maintainability. Clear boundaries and responsibilities.

## Related

- [cqrs-pattern.md](./cqrs-pattern.md) - Command Query Responsibility Segregation
- [domain-driven-design.md](./domain-driven-design.md) - DDD patterns
- [dependency-injection.md](../03-csharp/03-dotnet/dependency-injection.md) - DI patterns
