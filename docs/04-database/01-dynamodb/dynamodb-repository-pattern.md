# DynamoDB Repository Pattern

Repository pattern implementation for DynamoDB. Encapsulate data access logic and provide clean interface for business layer.

## Principle

Repositories abstract DynamoDB operations. Business logic depends on interfaces, not concrete implementations. Enable testability and maintainability.

## Repository Interface

Define repository contract:

```csharp
// LibraryService.Domain/Interfaces/IUserRepository.cs
namespace LibraryService.Domain.Interfaces;

public interface IUserRepository
{
    Task<User?> GetByIdAsync(string userId);
    Task<User?> GetByEmailAsync(string email);
    Task<User[]> GetAllAsync(int limit = 100);
    Task SaveAsync(User user);
    Task DeleteAsync(string userId);
}
```

## Base Repository

Create base class for common DynamoDB operations:

```csharp
// LibraryService.Infrastructure/Data/DynamoDbRepositoryBase.cs
namespace LibraryService.Infrastructure.Data;

public abstract class DynamoDbRepositoryBase
{
    protected readonly IAmazonDynamoDB DynamoDb;
    protected readonly string TableName;
    protected readonly ILogger Logger;

    protected DynamoDbRepositoryBase(
        IAmazonDynamoDB dynamoDb,
        IOptions<DynamoDbOptions> options,
        ILogger logger)
    {
        DynamoDb = dynamoDb;
        TableName = options.Value.TableName;
        Logger = logger;
    }

    protected async Task<Dictionary<string, AttributeValue>?> GetItemAsync(
        string pk,
        string sk,
        CancellationToken ct = default)
    {
        var request = new GetItemRequest
        {
            TableName = TableName,
            Key = new Dictionary<string, AttributeValue>
            {
                ["PK"] = new AttributeValue { S = pk },
                ["SK"] = new AttributeValue { S = sk }
            }
        };

        var response = await DynamoDb.GetItemAsync(request, ct);

        if (response.Item.Count == 0)
        {
            Logger.LogDebug("Item not found: PK={PK}, SK={SK}", pk, sk);
            return null;
        }

        return response.Item;
    }

    protected async Task<List<Dictionary<string, AttributeValue>>> QueryAsync(
        string pk,
        string? skPrefix = null,
        int limit = 100,
        CancellationToken ct = default)
    {
        var request = new QueryRequest
        {
            TableName = TableName,
            KeyConditionExpression = skPrefix != null
                ? "PK = :pk AND begins_with(SK, :sk)"
                : "PK = :pk",
            ExpressionAttributeValues = new Dictionary<string, AttributeValue>
            {
                [":pk"] = new AttributeValue { S = pk }
            },
            Limit = limit
        };

        if (skPrefix != null)
        {
            request.ExpressionAttributeValues[":sk"] = new AttributeValue { S = skPrefix };
        }

        var response = await DynamoDb.QueryAsync(request, ct);
        return response.Items;
    }

    protected async Task<List<Dictionary<string, AttributeValue>>> QueryIndexAsync(
        string indexName,
        string pk,
        string? sk = null,
        int limit = 100,
        CancellationToken ct = default)
    {
        var request = new QueryRequest
        {
            TableName = TableName,
            IndexName = indexName,
            KeyConditionExpression = sk != null
                ? "GSI1PK = :pk AND GSI1SK = :sk"
                : "GSI1PK = :pk",
            ExpressionAttributeValues = new Dictionary<string, AttributeValue>
            {
                [":pk"] = new AttributeValue { S = pk }
            },
            Limit = limit
        };

        if (sk != null)
        {
            request.ExpressionAttributeValues[":sk"] = new AttributeValue { S = sk };
        }

        var response = await DynamoDb.QueryAsync(request, ct);
        return response.Items;
    }

    protected async Task PutItemAsync(
        Dictionary<string, AttributeValue> item,
        CancellationToken ct = default)
    {
        var request = new PutItemRequest
        {
            TableName = TableName,
            Item = item
        };

        await DynamoDb.PutItemAsync(request, ct);
        Logger.LogDebug("Item saved: PK={PK}, SK={SK}", item["PK"].S, item["SK"].S);
    }

    protected async Task DeleteItemAsync(
        string pk,
        string sk,
        CancellationToken ct = default)
    {
        var request = new DeleteItemRequest
        {
            TableName = TableName,
            Key = new Dictionary<string, AttributeValue>
            {
                ["PK"] = new AttributeValue { S = pk },
                ["SK"] = new AttributeValue { S = sk }
            }
        };

        await DynamoDb.DeleteItemAsync(request, ct);
        Logger.LogDebug("Item deleted: PK={PK}, SK={SK}", pk, sk);
    }

    protected async Task<bool> ItemExistsAsync(
        string pk,
        string sk,
        CancellationToken ct = default)
    {
        var item = await GetItemAsync(pk, sk, ct);
        return item != null;
    }
}
```

## User Repository Implementation

Concrete repository with mapping:

```csharp
// LibraryService.Infrastructure/Repositories/UserRepository.cs
namespace LibraryService.Infrastructure.Repositories;

public class UserRepository : DynamoDbRepositoryBase, IUserRepository
{
    public UserRepository(
        IAmazonDynamoDB dynamoDb,
        IOptions<DynamoDbOptions> options,
        ILogger<UserRepository> logger)
        : base(dynamoDb, options, logger)
    {
    }

    public async Task<User?> GetByIdAsync(string userId)
    {
        var item = await GetItemAsync($"USER#{userId}", $"USER#{userId}");
        return item != null ? MapToUser(item) : null;
    }

    public async Task<User?> GetByEmailAsync(string email)
    {
        var items = await QueryIndexAsync(
            indexName: "GSI1",
            pk: $"EMAIL#{email}");

        return items.Count > 0 ? MapToUser(items[0]) : null;
    }

    public async Task<User[]> GetAllAsync(int limit = 100)
    {
        // Query all users (requires GSI or Scan)
        var items = await QueryIndexAsync(
            indexName: "GSI1",
            pk: "USERS",
            limit: limit);

        return items.Select(MapToUser).ToArray();
    }

    public async Task SaveAsync(User user)
    {
        var item = new Dictionary<string, AttributeValue>
        {
            ["PK"] = new AttributeValue { S = $"USER#{user.Id}" },
            ["SK"] = new AttributeValue { S = $"USER#{user.Id}" },
            ["EntityType"] = new AttributeValue { S = "User" },
            ["UserId"] = new AttributeValue { S = user.Id },
            ["Email"] = new AttributeValue { S = user.Email },
            ["Name"] = new AttributeValue { S = user.Name },
            ["CreatedAt"] = new AttributeValue { S = user.CreatedAt.ToString("o") },
            ["UpdatedAt"] = new AttributeValue { S = DateTime.UtcNow.ToString("o") },
            // GSI for email lookup
            ["GSI1PK"] = new AttributeValue { S = $"EMAIL#{user.Email}" },
            ["GSI1SK"] = new AttributeValue { S = $"USER#{user.Id}" }
        };

        await PutItemAsync(item);
    }

    public async Task DeleteAsync(string userId)
    {
        await DeleteItemAsync($"USER#{userId}", $"USER#{userId}");
    }

    private User MapToUser(Dictionary<string, AttributeValue> item)
    {
        return new User
        {
            Id = item["UserId"].S,
            Email = item["Email"].S,
            Name = item["Name"].S,
            CreatedAt = DateTime.Parse(item["CreatedAt"].S)
        };
    }
}
```

## Complex Repository with Relations

Handle entities with relationships:

```csharp
// LibraryService.Infrastructure/Repositories/LoanRepository.cs
namespace LibraryService.Infrastructure.Repositories;

public class LoanRepository : DynamoDbRepositoryBase, ILoanRepository
{
    public LoanRepository(
        IAmazonDynamoDB dynamoDb,
        IOptions<DynamoDbOptions> options,
        ILogger<LoanRepository> logger)
        : base(dynamoDb, options, logger)
    {
    }

    public async Task<Loan?> GetByIdAsync(string loanId)
    {
        // Loans are stored under both user and book
        // Query from user perspective (requires knowing userId)
        // Alternative: Use GSI with LOAN# prefix
        var items = await QueryIndexAsync(
            indexName: "GSI2",
            pk: $"LOAN#{loanId}");

        return items.Count > 0 ? MapToLoan(items[0]) : null;
    }

    public async Task<Loan[]> GetByUserIdAsync(string userId)
    {
        var items = await QueryAsync($"USER#{userId}", "LOAN#");
        return items.Select(MapToLoan).ToArray();
    }

    public async Task<Loan[]> GetByBookIdAsync(string bookId)
    {
        var items = await QueryAsync($"BOOK#{bookId}", "LOAN#");
        return items.Select(MapToLoan).ToArray();
    }

    public async Task<Loan[]> GetActiveLoansAsync(int limit = 100)
    {
        var items = await QueryIndexAsync(
            indexName: "GSI2",
            pk: "LOAN#ACTIVE",
            limit: limit);

        return items.Select(MapToLoan).ToArray();
    }

    public async Task SaveAsync(Loan loan)
    {
        // Store under user partition
        var userItem = CreateLoanItem($"USER#{loan.UserId}", loan);

        // Store under book partition
        var bookItem = CreateLoanItem($"BOOK#{loan.BookId}", loan);

        // Use transaction for atomicity
        var request = new TransactWriteItemsRequest
        {
            TransactItems = new List<TransactWriteItem>
            {
                new TransactWriteItem
                {
                    Put = new Put
                    {
                        TableName = TableName,
                        Item = userItem
                    }
                },
                new TransactWriteItem
                {
                    Put = new Put
                    {
                        TableName = TableName,
                        Item = bookItem
                    }
                }
            }
        };

        await DynamoDb.TransactWriteItemsAsync(request);
        Logger.LogInformation("Loan saved: {LoanId}", loan.Id);
    }

    public async Task DeleteAsync(string loanId)
    {
        // Need to delete from both user and book partitions
        // Requires knowing userId and bookId (fetch first or store in GSI)
        var loan = await GetByIdAsync(loanId);
        if (loan == null) return;

        var request = new TransactWriteItemsRequest
        {
            TransactItems = new List<TransactWriteItem>
            {
                new TransactWriteItem
                {
                    Delete = new Delete
                    {
                        TableName = TableName,
                        Key = new Dictionary<string, AttributeValue>
                        {
                            ["PK"] = new AttributeValue { S = $"USER#{loan.UserId}" },
                            ["SK"] = new AttributeValue { S = $"LOAN#{loanId}" }
                        }
                    }
                },
                new TransactWriteItem
                {
                    Delete = new Delete
                    {
                        TableName = TableName,
                        Key = new Dictionary<string, AttributeValue>
                        {
                            ["PK"] = new AttributeValue { S = $"BOOK#{loan.BookId}" },
                            ["SK"] = new AttributeValue { S = $"LOAN#{loanId}" }
                        }
                    }
                }
            }
        };

        await DynamoDb.TransactWriteItemsAsync(request);
    }

    private Dictionary<string, AttributeValue> CreateLoanItem(string pk, Loan loan)
    {
        var item = new Dictionary<string, AttributeValue>
        {
            ["PK"] = new AttributeValue { S = pk },
            ["SK"] = new AttributeValue { S = $"LOAN#{loan.Id}" },
            ["EntityType"] = new AttributeValue { S = "Loan" },
            ["LoanId"] = new AttributeValue { S = loan.Id },
            ["UserId"] = new AttributeValue { S = loan.UserId },
            ["BookId"] = new AttributeValue { S = loan.BookId },
            ["DueDate"] = new AttributeValue { S = loan.DueDate.ToString("o") },
            ["Status"] = new AttributeValue { S = loan.Status.ToString() },
            ["CreatedAt"] = new AttributeValue { S = loan.CreatedAt.ToString("o") }
        };

        // GSI for querying by loan ID
        item["GSI2PK"] = new AttributeValue { S = $"LOAN#{loan.Id}" };
        item["GSI2SK"] = new AttributeValue { S = loan.CreatedAt.ToString("o") };

        // GSI for active loans query
        if (loan.Status == LoanStatus.Active)
        {
            item["GSI3PK"] = new AttributeValue { S = "LOAN#ACTIVE" };
            item["GSI3SK"] = new AttributeValue { S = loan.DueDate.ToString("o") };
        }

        return item;
    }

    private Loan MapToLoan(Dictionary<string, AttributeValue> item)
    {
        return new Loan
        {
            Id = item["LoanId"].S,
            UserId = item["UserId"].S,
            BookId = item["BookId"].S,
            DueDate = DateTime.Parse(item["DueDate"].S),
            Status = Enum.Parse<LoanStatus>(item["Status"].S),
            CreatedAt = DateTime.Parse(item["CreatedAt"].S)
        };
    }
}
```

## Error Handling

Add retry logic and error handling:

```csharp
public class ResilientUserRepository : IUserRepository
{
    private readonly UserRepository _inner;
    private readonly ILogger<ResilientUserRepository> _logger;

    public ResilientUserRepository(
        UserRepository inner,
        ILogger<ResilientUserRepository> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<User?> GetByIdAsync(string userId)
    {
        return await RetryAsync(
            () => _inner.GetByIdAsync(userId),
            maxRetries: 3);
    }

    public async Task SaveAsync(User user)
    {
        await RetryAsync(
            () => _inner.SaveAsync(user),
            maxRetries: 3);
    }

    private async Task<T> RetryAsync<T>(
        Func<Task<T>> operation,
        int maxRetries)
    {
        for (int retry = 0; retry <= maxRetries; retry++)
        {
            try
            {
                return await operation();
            }
            catch (ProvisionedThroughputExceededException ex) when (retry < maxRetries)
            {
                var delay = TimeSpan.FromMilliseconds(Math.Pow(2, retry) * 100);
                _logger.LogWarning(ex,
                    "Throughput exceeded, retrying in {Delay}ms (attempt {Retry}/{MaxRetries})",
                    delay.TotalMilliseconds, retry + 1, maxRetries);
                await Task.Delay(delay);
            }
            catch (ThrottlingException ex) when (retry < maxRetries)
            {
                var delay = TimeSpan.FromMilliseconds(Math.Pow(2, retry) * 100);
                _logger.LogWarning(ex,
                    "Request throttled, retrying in {Delay}ms (attempt {Retry}/{MaxRetries})",
                    delay.TotalMilliseconds, retry + 1, maxRetries);
                await Task.Delay(delay);
            }
        }

        throw new Exception($"Max retries ({maxRetries}) exceeded");
    }
}
```

## Dependency Registration

Register repositories in DI container:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// DynamoDB client
builder.Services.AddSingleton<IAmazonDynamoDB>(sp =>
{
    var config = new AmazonDynamoDBConfig
    {
        RegionEndpoint = RegionEndpoint.USEast1
    };
    return new AmazonDynamoDBClient(config);
});

// Configuration
builder.Services.Configure<DynamoDbOptions>(
    builder.Configuration.GetSection(DynamoDbOptions.SectionName));

// Repositories
builder.Services.AddScoped<UserRepository>();
builder.Services.AddScoped<IUserRepository, ResilientUserRepository>();
builder.Services.AddScoped<LoanRepository>();
builder.Services.AddScoped<ILoanRepository, LoanRepository>();
```

Configuration:

```json
{
  "DynamoDb": {
    "TableName": "library-service-prod"
  }
}
```

## Testing Repositories

Unit test with mocked DynamoDB:

```csharp
public class UserRepositoryTests
{
    private readonly IAmazonDynamoDB _dynamoDb;
    private readonly IOptions<DynamoDbOptions> _options;
    private readonly ILogger<UserRepository> _logger;
    private readonly UserRepository _repository;

    public UserRepositoryTests()
    {
        _dynamoDb = Substitute.For<IAmazonDynamoDB>();
        _options = Options.Create(new DynamoDbOptions { TableName = "test-table" });
        _logger = Substitute.For<ILogger<UserRepository>>();
        _repository = new UserRepository(_dynamoDb, _options, _logger);
    }

    [Fact]
    public async Task GetByIdAsync_ExistingUser_ReturnsUser()
    {
        // Arrange
        var userId = "user_123";
        var item = new Dictionary<string, AttributeValue>
        {
            ["UserId"] = new AttributeValue { S = userId },
            ["Email"] = new AttributeValue { S = "john@example.com" },
            ["Name"] = new AttributeValue { S = "John Doe" },
            ["CreatedAt"] = new AttributeValue { S = DateTime.UtcNow.ToString("o") }
        };

        _dynamoDb.GetItemAsync(Arg.Any<GetItemRequest>(), Arg.Any<CancellationToken>())
            .Returns(new GetItemResponse { Item = item });

        // Act
        var user = await _repository.GetByIdAsync(userId);

        // Assert
        Assert.NotNull(user);
        Assert.Equal(userId, user.Id);
        Assert.Equal("john@example.com", user.Email);
    }

    [Fact]
    public async Task SaveAsync_NewUser_CallsPutItem()
    {
        // Arrange
        var user = new User
        {
            Id = "user_123",
            Email = "john@example.com",
            Name = "John Doe",
            CreatedAt = DateTime.UtcNow
        };

        // Act
        await _repository.SaveAsync(user);

        // Assert
        await _dynamoDb.Received(1).PutItemAsync(
            Arg.Is<PutItemRequest>(r =>
                r.TableName == "test-table" &&
                r.Item["PK"].S == "USER#user_123" &&
                r.Item["Email"].S == "john@example.com"),
            Arg.Any<CancellationToken>());
    }
}
```

Integration test with local DynamoDB:

```csharp
public class UserRepositoryIntegrationTests : IAsyncLifetime
{
    private readonly IAmazonDynamoDB _dynamoDb;
    private readonly UserRepository _repository;
    private readonly string _tableName = "test-users";

    public UserRepositoryIntegrationTests()
    {
        _dynamoDb = new AmazonDynamoDBClient(new AmazonDynamoDBConfig
        {
            ServiceURL = "http://localhost:8000"  // DynamoDB Local
        });

        var options = Options.Create(new DynamoDbOptions { TableName = _tableName });
        var logger = Substitute.For<ILogger<UserRepository>>();
        _repository = new UserRepository(_dynamoDb, options, logger);
    }

    public async Task InitializeAsync()
    {
        // Create table
        await _dynamoDb.CreateTableAsync(new CreateTableRequest
        {
            TableName = _tableName,
            KeySchema = new List<KeySchemaElement>
            {
                new KeySchemaElement { AttributeName = "PK", KeyType = KeyType.HASH },
                new KeySchemaElement { AttributeName = "SK", KeyType = KeyType.RANGE }
            },
            AttributeDefinitions = new List<AttributeDefinition>
            {
                new AttributeDefinition { AttributeName = "PK", AttributeType = ScalarAttributeType.S },
                new AttributeDefinition { AttributeName = "SK", AttributeType = ScalarAttributeType.S }
            },
            BillingMode = BillingMode.PAY_PER_REQUEST
        });
    }

    public async Task DisposeAsync()
    {
        // Delete table
        await _dynamoDb.DeleteTableAsync(_tableName);
    }

    [Fact]
    public async Task SaveAndRetrieve_User_RoundTrip()
    {
        // Arrange
        var user = new User
        {
            Id = Guid.NewGuid().ToString(),
            Email = "integration@example.com",
            Name = "Integration Test",
            CreatedAt = DateTime.UtcNow
        };

        // Act
        await _repository.SaveAsync(user);
        var retrieved = await _repository.GetByIdAsync(user.Id);

        // Assert
        Assert.NotNull(retrieved);
        Assert.Equal(user.Id, retrieved.Id);
        Assert.Equal(user.Email, retrieved.Email);
        Assert.Equal(user.Name, retrieved.Name);
    }
}
```

## Guidelines

**Repository Design:**
- One interface per aggregate root
- Keep repositories focused on data access
- No business logic in repositories
- Return domain entities, not DynamoDB items

**Base Repository:**
- Extract common operations (GetItem, Query, PutItem)
- Handle logging and error handling centrally
- Parameterize table name and configuration
- Provide protected methods for derived classes

**Mapping:**
- Separate mapping logic from queries
- Use private methods for ToEntity and ToItem
- Handle null values gracefully
- Convert dates to ISO 8601 strings

**Transactions:**
- Use for multi-item operations requiring atomicity
- Limit to 100 items per transaction
- Handle transaction failures with retries
- Log transaction operations

**Error Handling:**
- Retry throttling and throughput exceptions
- Log errors with context (PK, SK, operation)
- Wrap exceptions with meaningful messages
- Use decorator pattern for resilience

## Benefits

Testability. Mock repositories in unit tests.

Maintainability. Centralize data access logic.

Flexibility. Swap implementations easily.

Clean architecture. Business logic independent of database.

## Related

- [single-table-design.md](./single-table-design.md) - Table structure
- [dynamodb-best-practices.md](./dynamodb-best-practices.md) - Optimization patterns
- [repository-pattern.md](../../07-architecture/repository-pattern.md) - Generic pattern
