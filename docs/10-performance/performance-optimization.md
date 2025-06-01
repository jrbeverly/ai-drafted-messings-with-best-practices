# Performance Optimization

Optimize application performance. Database queries, async operations, memory management.

## Principle

Measure first. Optimize bottlenecks. Avoid premature optimization. Profile in production.

## Async/Await Best Practices

Use async all the way:

```csharp
// GOOD - Async all the way
public async Task<User> GetUserAsync(string userId)
{
    var user = await _repository.GetByIdAsync(userId);
    var loans = await _loanRepository.GetByUserIdAsync(userId);

    return new UserWithLoans
    {
        User = user,
        Loans = loans
    };
}

// BAD - Blocking async code
public User GetUser(string userId)
{
    var user = _repository.GetByIdAsync(userId).Result;  // Deadlock risk!
    return user;
}

// BAD - Unnecessary async
public async Task<string> GetName()
{
    return await Task.FromResult("John");  // Pointless async
}

// GOOD - Synchronous when appropriate
public string GetName()
{
    return "John";
}
```

Parallel operations:

```csharp
// Sequential (slow)
public async Task<UserDetails> GetUserDetailsSlowAsync(string userId)
{
    var user = await _userRepository.GetByIdAsync(userId);  // 100ms
    var loans = await _loanRepository.GetByUserIdAsync(userId);  // 100ms
    var settings = await _settingsRepository.GetByUserIdAsync(userId);  // 100ms
    // Total: 300ms
}

// Parallel (fast)
public async Task<UserDetails> GetUserDetailsFastAsync(string userId)
{
    var userTask = _userRepository.GetByIdAsync(userId);
    var loansTask = _loanRepository.GetByUserIdAsync(userId);
    var settingsTask = _settingsRepository.GetByUserIdAsync(userId);

    await Task.WhenAll(userTask, loansTask, settingsTask);

    return new UserDetails
    {
        User = userTask.Result,
        Loans = loansTask.Result,
        Settings = settingsTask.Result
    };
    // Total: 100ms
}
```

## Database Query Optimization

Efficient queries:

```csharp
// BAD - N+1 query problem
public async Task<User[]> GetUsersWithLoansSlowAsync()
{
    var users = await _userRepository.GetAllAsync();

    foreach (var user in users)
    {
        user.Loans = await _loanRepository.GetByUserIdAsync(user.Id);  // N queries!
    }

    return users;
}

// GOOD - Batch query
public async Task<User[]> GetUsersWithLoansFastAsync()
{
    var users = await _userRepository.GetAllAsync();
    var userIds = users.Select(u => u.Id).ToArray();

    // Single batch query for all loans
    var allLoans = await _loanRepository.GetByUserIdsAsync(userIds);

    var loansByUser = allLoans.GroupBy(l => l.UserId)
        .ToDictionary(g => g.Key, g => g.ToArray());

    foreach (var user in users)
    {
        user.Loans = loansByUser.GetValueOrDefault(user.Id) ?? Array.Empty<Loan>();
    }

    return users;
}
```

DynamoDB batch operations:

```csharp
// GOOD - Batch get (up to 100 items)
public async Task<User[]> GetUsersByIdsAsync(string[] userIds)
{
    var keys = userIds.Select(id => new Dictionary<string, AttributeValue>
    {
        ["PK"] = new AttributeValue { S = $"USER#{id}" },
        ["SK"] = new AttributeValue { S = $"USER#{id}" }
    }).ToList();

    var request = new BatchGetItemRequest
    {
        RequestItems = new Dictionary<string, KeysAndAttributes>
        {
            [_tableName] = new KeysAndAttributes { Keys = keys }
        }
    };

    var response = await _dynamoDb.BatchGetItemAsync(request);

    return response.Responses[_tableName]
        .Select(MapToUser)
        .ToArray();
}
```

Projection to reduce data transfer:

```csharp
// BAD - Fetch entire item
public async Task<UserSummary[]> GetUserSummariesSlowAsync()
{
    var users = await _userRepository.GetAllAsync();  // Gets all fields

    return users.Select(u => new UserSummary
    {
        Id = u.Id,
        Name = u.Name,
        Email = u.Email
        // Only need 3 fields, but fetched everything
    }).ToArray();
}

// GOOD - Projection expression
public async Task<UserSummary[]> GetUserSummariesFastAsync()
{
    var request = new ScanRequest
    {
        TableName = _tableName,
        ProjectionExpression = "UserId, #name, Email",  // Only fetch needed fields
        ExpressionAttributeNames = new Dictionary<string, string>
        {
            ["#name"] = "Name"
        }
    };

    var response = await _dynamoDb.ScanAsync(request);

    return response.Items.Select(item => new UserSummary
    {
        Id = item["UserId"].S,
        Name = item["Name"].S,
        Email = item["Email"].S
    }).ToArray();
}
```

## Memory Optimization

Avoid unnecessary allocations:

```csharp
// BAD - Multiple allocations
public string FormatUserListSlow(User[] users)
{
    var result = "";
    foreach (var user in users)
    {
        result += $"{user.Name}, ";  // Creates new string each iteration!
    }
    return result.TrimEnd(',', ' ');
}

// GOOD - StringBuilder
public string FormatUserListFast(User[] users)
{
    var sb = new StringBuilder();
    foreach (var user in users)
    {
        sb.Append(user.Name);
        sb.Append(", ");
    }
    if (sb.Length > 0)
        sb.Length -= 2;  // Remove last ", "

    return sb.ToString();
}
```

Span<T> for zero-allocation parsing:

```csharp
// BAD - Allocates substrings
public (string prefix, string id) ParseUserIdSlow(string userId)
{
    var parts = userId.Split('_');  // Allocates array
    return (parts[0], parts[1]);    // Allocates strings
}

// GOOD - Span<char> (zero allocations)
public (ReadOnlySpan<char> prefix, ReadOnlySpan<char> id) ParseUserIdFast(string userId)
{
    var span = userId.AsSpan();
    var separatorIndex = span.IndexOf('_');

    if (separatorIndex < 0)
        return (span, ReadOnlySpan<char>.Empty);

    return (span.Slice(0, separatorIndex), span.Slice(separatorIndex + 1));
}
```

ArrayPool for large arrays:

```csharp
public async Task<byte[]> ProcessLargeData()
{
    var buffer = ArrayPool<byte>.Shared.Rent(4096);

    try
    {
        // Use buffer
        await ReadDataIntoBufferAsync(buffer);
        return ProcessBuffer(buffer);
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

## JSON Serialization

System.Text.Json optimization:

```csharp
// Configure for performance
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    options.SerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
    options.SerializerOptions.WriteIndented = false;  // Smaller payload

    // Source generation for AOT
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonSerializerContext.Default);
});

// Source-generated serializer context
[JsonSerializable(typeof(User))]
[JsonSerializable(typeof(User[]))]
[JsonSerializable(typeof(Loan))]
[JsonSerializable(typeof(ApiResponse<User>))]
internal partial class AppJsonSerializerContext : JsonSerializerContext
{
}
```

## Lazy Loading

Defer expensive operations:

```csharp
public class UserDetails
{
    private readonly ILoanRepository _loanRepository;
    private Loan[]? _loans;

    public required User User { get; init; }

    public async Task<Loan[]> GetLoansAsync()
    {
        // Lazy load loans only when accessed
        if (_loans == null)
        {
            _loans = await _loanRepository.GetByUserIdAsync(User.Id);
        }

        return _loans;
    }
}

// Or use Lazy<T>
public class ExpensiveService
{
    private readonly Lazy<HeavyObject> _heavy = new(() => new HeavyObject());

    public HeavyObject Heavy => _heavy.Value;  // Only created when accessed
}
```

## Response Compression

Enable compression:

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
    options.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
        new[] { "application/json" });
});

builder.Services.Configure<BrotliCompressionProviderOptions>(options =>
{
    options.Level = CompressionLevel.Fastest;
});

app.UseResponseCompression();
```

## HTTP Client Optimization

Reuse HttpClient:

```csharp
// BAD - Creates new client every call
public async Task<string> CallApiSlowAsync()
{
    using var client = new HttpClient();  // Socket exhaustion!
    return await client.GetStringAsync("https://api.example.com/data");
}

// GOOD - Use IHttpClientFactory
public class ExternalApiService
{
    private readonly HttpClient _httpClient;

    public ExternalApiService(IHttpClientFactory httpClientFactory)
    {
        _httpClient = httpClientFactory.CreateClient("ExternalApi");
    }

    public async Task<string> CallApiAsync()
    {
        return await _httpClient.GetStringAsync("/data");
    }
}

// Register
builder.Services.AddHttpClient("ExternalApi", client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
    client.Timeout = TimeSpan.FromSeconds(30);
    client.DefaultRequestHeaders.Add("User-Agent", "LibraryService/1.0");
});
```

Connection pooling:

```csharp
builder.Services.AddHttpClient("ExternalApi")
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
    {
        PooledConnectionLifetime = TimeSpan.FromMinutes(5),
        PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2),
        MaxConnectionsPerServer = 20
    });
```

## Pagination

Always paginate large results:

```csharp
public record PagedRequest
{
    [Range(1, int.MaxValue)]
    public int Page { get; init; } = 1;

    [Range(1, 100)]
    public int Limit { get; init; } = 20;
}

public async Task<PagedResponse<User>> GetUsersPagedAsync(PagedRequest request)
{
    // Calculate offset
    var offset = (request.Page - 1) * request.Limit;

    // Fetch one extra to check if there are more results
    var users = await _repository.GetUsersAsync(offset, request.Limit + 1);

    var hasMore = users.Length > request.Limit;
    var items = hasMore ? users[..request.Limit] : users;

    return new PagedResponse<User>
    {
        Items = items,
        Page = request.Page,
        Limit = request.Limit,
        HasMore = hasMore
    };
}
```

## Benchmarking

BenchmarkDotNet:

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
public class UserServiceBenchmark
{
    private UserService _service;
    private string _userId;

    [GlobalSetup]
    public void Setup()
    {
        _service = new UserService(/* ... */);
        _userId = "user_123";
    }

    [Benchmark]
    public async Task GetUserAsync()
    {
        await _service.GetUserAsync(_userId);
    }

    [Benchmark]
    public async Task GetUserWithLoansAsync()
    {
        await _service.GetUserWithLoansAsync(_userId);
    }
}

// Run benchmarks
public class Program
{
    public static void Main(string[] args)
    {
        BenchmarkRunner.Run<UserServiceBenchmark>();
    }
}
```

## Profiling

MiniProfiler integration:

```csharp
builder.Services.AddMiniProfiler(options =>
{
    options.RouteBasePath = "/profiler";
    options.ColorScheme = ColorScheme.Auto;
});

app.UseMiniProfiler();

// Profile code sections
public async Task<User> GetUserAsync(string userId)
{
    using (MiniProfiler.Current.Step("Get user from database"))
    {
        var user = await _repository.GetByIdAsync(userId);

        using (MiniProfiler.Current.Step("Get user loans"))
        {
            user.Loans = await _loanRepository.GetByUserIdAsync(userId);
        }

        return user;
    }
}
```

## Lambda Cold Start Optimization

Reduce cold start time:

```csharp
// Initialize static dependencies outside handler
public class Function
{
    private static readonly IAmazonDynamoDB DynamoDb = new AmazonDynamoDBClient();
    private static readonly HttpClient HttpClient = new HttpClient();

    private readonly IUserService _userService;

    // Constructor runs once per container
    public Function()
    {
        var services = new ServiceCollection();
        services.AddSingleton(DynamoDb);
        services.AddSingleton(HttpClient);
        services.AddScoped<IUserService, UserService>();

        var provider = services.BuildServiceProvider();
        _userService = provider.GetRequiredService<IUserService>();
    }

    // Handler runs per invocation
    public async Task<APIGatewayProxyResponse> FunctionHandler(
        APIGatewayProxyRequest request,
        ILambdaContext context)
    {
        var result = await _userService.GetUserAsync(userId);
        return CreateResponse(result);
    }
}
```

Native AOT compilation:

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
  <StripSymbols>true</StripSymbols>
  <InvariantGlobalization>true</InvariantGlobalization>
</PropertyGroup>
```

## Guidelines

**General:**
- Measure before optimizing
- Profile in production-like environment
- Optimize bottlenecks first
- Monitor performance metrics

**Async:**
- Use async/await consistently
- Avoid blocking on async code
- Parallelize independent operations
- Don't create unnecessary async methods

**Database:**
- Batch queries when possible
- Use projections to fetch only needed data
- Avoid N+1 queries
- Implement pagination for lists

**Memory:**
- Avoid unnecessary allocations
- Use StringBuilder for string concatenation
- Use Span<T> for zero-allocation parsing
- ArrayPool for large temporary arrays

**HTTP:**
- Use IHttpClientFactory
- Configure connection pooling
- Enable response compression
- Reuse connections

**Caching:**
- Cache expensive operations
- Set appropriate TTL
- Monitor cache hit rate
- Invalidate on write

## Benefits

Performance. Faster response times.

Scalability. Handle more concurrent requests.

Cost. Lower resource consumption.

User experience. Responsive application.

## Related

- [caching-strategies.md](./caching-strategies.md) - Caching patterns
- [monitoring-observability.md](./monitoring-observability.md) - Performance monitoring
- [lambda-best-practices.md](../05-infrastructure/02-aws-lambda/lambda-best-practices.md) - Lambda optimization
