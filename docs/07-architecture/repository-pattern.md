# Repository Pattern

Abstract data access behind repository interfaces. Decouple business logic from persistence details.

## Principle

Repositories encapsulate data access. Business logic works with domain entities, not database details. Swap implementations without changing business code.

## Basic Repository

Define interface in domain:

```csharp
// LibraryService.Domain/Interfaces/IUserRepository.cs
public interface IUserRepository
{
    Task<User?> GetByIdAsync(string id);
    Task<User[]> GetAllAsync();
    Task SaveAsync(User user);
    Task DeleteAsync(string id);
}
```

Implement in infrastructure:

```csharp
// LibraryService.Infrastructure/Repositories/UserRepository.cs
public class UserRepository : IUserRepository
{
    private readonly IAmazonDynamoDB _dynamoDb;
    private readonly string _tableName;

    public UserRepository(IAmazonDynamoDB dynamoDb, IOptions<DatabaseOptions> options)
    {
        _dynamoDb = dynamoDb;
        _tableName = options.Value.TableName;
    }

    public async Task<User?> GetByIdAsync(string id)
    {
        var request = new GetItemRequest
        {
            TableName = _tableName,
            Key = new Dictionary<string, AttributeValue>
            {
                ["PK"] = new AttributeValue { S = $"USER#{id}" },
                ["SK"] = new AttributeValue { S = $"USER#{id}" }
            }
        };

        var response = await _dynamoDb.GetItemAsync(request);

        return response.Item.Count > 0
            ? MapToDomain(response.Item)
            : null;
    }

    public async Task SaveAsync(User user)
    {
        var request = new PutItemRequest
        {
            TableName = _tableName,
            Item = MapToItem(user)
        };

        await _dynamoDb.PutItemAsync(request);
    }

    private User MapToDomain(Dictionary<string, AttributeValue> item)
    {
        return new User
        {
            Id = item["UserId"].S,
            Email = item["Email"].S,
            Name = item["Name"].S,
            IsActive = item["IsActive"].BOOL,
            CreatedAt = DateTime.Parse(item["CreatedAt"].S)
        };
    }

    private Dictionary<string, AttributeValue> MapToItem(User user)
    {
        return new Dictionary<string, AttributeValue>
        {
            ["PK"] = new AttributeValue { S = $"USER#{user.Id}" },
            ["SK"] = new AttributeValue { S = $"USER#{user.Id}" },
            ["UserId"] = new AttributeValue { S = user.Id },
            ["Email"] = new AttributeValue { S = user.Email },
            ["Name"] = new AttributeValue { S = user.Name },
            ["IsActive"] = new AttributeValue { BOOL = user.IsActive },
            ["CreatedAt"] = new AttributeValue { S = user.CreatedAt.ToString("o") }
        };
    }
}
```

## Query Methods

Add specific query methods:

```csharp
public interface IUserRepository
{
    Task<User?> GetByIdAsync(string id);
    Task<User?> GetByEmailAsync(string email);
    Task<User[]> GetActiveUsersAsync();
    Task<User[]> SearchByNameAsync(string nameQuery);
    Task<User[]> GetByIdsAsync(string[] ids);
    Task SaveAsync(User user);
    Task DeleteAsync(string id);
}
```

Implementation:

```csharp
public async Task<User?> GetByEmailAsync(string email)
{
    var request = new QueryRequest
    {
        TableName = _tableName,
        IndexName = "EmailIndex",
        KeyConditionExpression = "Email = :email",
        ExpressionAttributeValues = new Dictionary<string, AttributeValue>
        {
            [":email"] = new AttributeValue { S = email }
        }
    };

    var response = await _dynamoDb.QueryAsync(request);

    return response.Items.Count > 0
        ? MapToDomain(response.Items[0])
        : null;
}

public async Task<User[]> GetActiveUsersAsync()
{
    var request = new ScanRequest
    {
        TableName = _tableName,
        FilterExpression = "IsActive = :true",
        ExpressionAttributeValues = new Dictionary<string, AttributeValue>
        {
            [":true"] = new AttributeValue { BOOL = true }
        }
    };

    var response = await _dynamoDb.ScanAsync(request);

    return response.Items.Select(MapToDomain).ToArray();
}
```

## Generic Repository

Base repository for common operations:

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(string id);
    Task<T[]> GetAllAsync();
    Task SaveAsync(T entity);
    Task DeleteAsync(string id);
}

public abstract class DynamoDbRepository<T> : IRepository<T> where T : class
{
    protected readonly IAmazonDynamoDB DynamoDb;
    protected readonly string TableName;

    protected DynamoDbRepository(
        IAmazonDynamoDB dynamoDb,
        IOptions<DatabaseOptions> options)
    {
        DynamoDb = dynamoDb;
        TableName = options.Value.TableName;
    }

    protected abstract string GetPartitionKey(string id);
    protected abstract string GetSortKey(string id);
    protected abstract T MapToDomain(Dictionary<string, AttributeValue> item);
    protected abstract Dictionary<string, AttributeValue> MapToItem(T entity);
    protected abstract string GetEntityId(T entity);

    public async Task<T?> GetByIdAsync(string id)
    {
        var request = new GetItemRequest
        {
            TableName = TableName,
            Key = new Dictionary<string, AttributeValue>
            {
                ["PK"] = new AttributeValue { S = GetPartitionKey(id) },
                ["SK"] = new AttributeValue { S = GetSortKey(id) }
            }
        };

        var response = await DynamoDb.GetItemAsync(request);

        return response.Item.Count > 0 ? MapToDomain(response.Item) : null;
    }

    public async Task SaveAsync(T entity)
    {
        var request = new PutItemRequest
        {
            TableName = TableName,
            Item = MapToItem(entity)
        };

        await DynamoDb.PutItemAsync(request);
    }
}
```

Specific repository extends base:

```csharp
public class UserRepository : DynamoDbRepository<User>, IUserRepository
{
    public UserRepository(
        IAmazonDynamoDB dynamoDb,
        IOptions<DatabaseOptions> options)
        : base(dynamoDb, options)
    {
    }

    protected override string GetPartitionKey(string id) => $"USER#{id}";
    protected override string GetSortKey(string id) => $"USER#{id}";
    protected override string GetEntityId(User entity) => entity.Id;

    protected override User MapToDomain(Dictionary<string, AttributeValue> item)
    {
        // Mapping logic
    }

    protected override Dictionary<string, AttributeValue> MapToItem(User entity)
    {
        // Mapping logic
    }

    // Add user-specific methods
    public async Task<User?> GetByEmailAsync(string email)
    {
        // Custom query
    }
}
```

## Specification Pattern

Query with specifications:

```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
}

public class ActiveUserSpecification : ISpecification<User>
{
    public bool IsSatisfiedBy(User entity)
    {
        return entity.IsActive;
    }
}

public interface IUserRepository
{
    Task<User[]> FindAsync(ISpecification<User> specification);
}

// Usage
var activeUsers = await _repository.FindAsync(new ActiveUserSpecification());
```

## Unit of Work

Coordinate multiple repositories:

```csharp
public interface IUnitOfWork : IDisposable
{
    IUserRepository Users { get; }
    ILoanRepository Loans { get; }
    IBookRepository Books { get; }

    Task<int> SaveChangesAsync();
    Task BeginTransactionAsync();
    Task CommitAsync();
    Task RollbackAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly IAmazonDynamoDB _dynamoDb;
    private readonly List<Action> _pendingOperations = new();

    public IUserRepository Users { get; }
    public ILoanRepository Loans { get; }
    public IBookRepository Books { get; }

    public UnitOfWork(
        IAmazonDynamoDB dynamoDb,
        IUserRepository users,
        ILoanRepository loans,
        IBookRepository books)
    {
        _dynamoDb = dynamoDb;
        Users = users;
        Loans = loans;
        Books = books;
    }

    public async Task<int> SaveChangesAsync()
    {
        // Execute all pending operations
        foreach (var operation in _pendingOperations)
        {
            operation();
        }

        _pendingOperations.Clear();
        return await Task.FromResult(_pendingOperations.Count);
    }

    public void Dispose()
    {
        _pendingOperations.Clear();
    }
}
```

## In-Memory Implementation

Testing implementation:

```csharp
public class InMemoryUserRepository : IUserRepository
{
    private readonly Dictionary<string, User> _users = new();

    public Task<User?> GetByIdAsync(string id)
    {
        _users.TryGetValue(id, out var user);
        return Task.FromResult(user);
    }

    public Task<User[]> GetAllAsync()
    {
        return Task.FromResult(_users.Values.ToArray());
    }

    public Task SaveAsync(User user)
    {
        _users[user.Id] = user;
        return Task.CompletedTask;
    }

    public Task DeleteAsync(string id)
    {
        _users.Remove(id);
        return Task.CompletedTask;
    }

    public Task<User?> GetByEmailAsync(string email)
    {
        var user = _users.Values.FirstOrDefault(u => u.Email == email);
        return Task.FromResult(user);
    }
}

// Use in tests
[Fact]
public async Task CreateUser_ValidData_SavesUser()
{
    var repository = new InMemoryUserRepository();
    var service = new UserService(repository);

    await service.CreateUserAsync("john@example.com", "John Doe");

    var users = await repository.GetAllAsync();
    Assert.Single(users);
}
```

## Pagination

Paginated queries:

```csharp
public record PagedResult<T>(
    T[] Items,
    int Total,
    int Page,
    int Limit,
    bool HasNextPage
);

public interface IUserRepository
{
    Task<PagedResult<User>> GetPagedAsync(int page, int limit);
}

public async Task<PagedResult<User>> GetPagedAsync(int page, int limit)
{
    var request = new ScanRequest
    {
        TableName = _tableName,
        Limit = limit
    };

    // For pagination beyond first page, need to track LastEvaluatedKey
    // Store in cache or pass as token

    var response = await _dynamoDb.ScanAsync(request);

    var items = response.Items.Select(MapToDomain).ToArray();
    var hasNextPage = response.LastEvaluatedKey.Count > 0;

    return new PagedResult<User>(
        items,
        items.Length, // DynamoDB doesn't provide total count easily
        page,
        limit,
        hasNextPage
    );
}
```

## Caching

Add caching layer:

```csharp
public class CachedUserRepository : IUserRepository
{
    private readonly IUserRepository _innerRepository;
    private readonly ICacheService _cache;

    public CachedUserRepository(
        IUserRepository innerRepository,
        ICacheService cache)
    {
        _innerRepository = innerRepository;
        _cache = cache;
    }

    public async Task<User?> GetByIdAsync(string id)
    {
        var cacheKey = $"user:{id}";

        var cached = _cache.Get<User>(cacheKey);
        if (cached != null)
            return cached;

        var user = await _innerRepository.GetByIdAsync(id);

        if (user != null)
            _cache.Set(cacheKey, user, TimeSpan.FromMinutes(5));

        return user;
    }

    public async Task SaveAsync(User user)
    {
        await _innerRepository.SaveAsync(user);

        // Invalidate cache
        var cacheKey = $"user:{user.Id}";
        _cache.Remove(cacheKey);
    }
}

// Register with decorator pattern
builder.Services.AddScoped<IUserRepository>(provider =>
{
    var innerRepo = new UserRepository(
        provider.GetRequiredService<IAmazonDynamoDB>(),
        provider.GetRequiredService<IOptions<DatabaseOptions>>()
    );

    var cache = provider.GetRequiredService<ICacheService>();

    return new CachedUserRepository(innerRepo, cache);
});
```

## Guidelines

**Interface Design:**
- Define in Domain layer
- Return domain entities
- No infrastructure types in signatures
- Specific methods over generic queries

**Implementation:**
- Implement in Infrastructure layer
- Handle mapping to/from storage
- Don't expose implementation details
- One repository per aggregate root

**Testing:**
- Create in-memory implementations
- Test business logic with mocks
- Test repository implementation separately

**When to Use:**
- Complex data access logic
- Multiple storage implementations
- Need for testability
- Clear domain boundaries

**When Not to Use:**
- Simple CRUD with no logic
- Single storage technology
- Overhead not justified

## Benefits

Abstraction. Decouple from storage details.

Testability. Easy to mock or use in-memory.

Flexibility. Swap implementations easily.

Encapsulation. Hide data access complexity.

## Related

- [clean-architecture.md](./clean-architecture.md) - Layered architecture
- [cqrs-pattern.md](./cqrs-pattern.md) - Read/write separation
- [unit-testing-best-practices.md](../03-csharp/02-testing/unit-testing-best-practices.md) - Testing repositories
