# Caching Strategies

Improve performance with caching. In-memory, distributed, and CDN caching patterns.

## Principle

Cache expensive operations. Invalidate appropriately. Set proper TTL. Monitor cache hit rate.

## In-Memory Caching

Use IMemoryCache for single-instance caching:

```csharp
public interface ICacheService
{
    T? Get<T>(string key);
    void Set<T>(string key, T value, TimeSpan? expiration = null);
    void Remove(string key);
    bool TryGet<T>(string key, out T? value);
}

public class MemoryCacheService : ICacheService
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<MemoryCacheService> _logger;

    public MemoryCacheService(IMemoryCache cache, ILogger<MemoryCacheService> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public T? Get<T>(string key)
    {
        return _cache.Get<T>(key);
    }

    public void Set<T>(string key, T value, TimeSpan? expiration = null)
    {
        var options = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = expiration ?? TimeSpan.FromMinutes(5),
            Size = 1  // For size-based eviction
        };

        _cache.Set(key, value, options);

        _logger.LogDebug("Cached item with key: {Key}", key);
    }

    public void Remove(string key)
    {
        _cache.Remove(key);
        _logger.LogDebug("Removed cached item with key: {Key}", key);
    }

    public bool TryGet<T>(string key, out T? value)
    {
        return _cache.TryGetValue(key, out value);
    }
}

// Register
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024;  // Limit cache size
});
builder.Services.AddSingleton<ICacheService, MemoryCacheService>();
```

Usage:

```csharp
public class UserService : IUserService
{
    private readonly IUserRepository _repository;
    private readonly ICacheService _cache;

    public async Task<User?> GetUserByIdAsync(string userId)
    {
        var cacheKey = $"user:{userId}";

        // Try cache first
        if (_cache.TryGet<User>(cacheKey, out var cachedUser))
        {
            return cachedUser;
        }

        // Cache miss - fetch from database
        var user = await _repository.GetByIdAsync(userId);

        if (user != null)
        {
            // Cache for 5 minutes
            _cache.Set(cacheKey, user, TimeSpan.FromMinutes(5));
        }

        return user;
    }

    public async Task UpdateUserAsync(string userId, UpdateUserRequest request)
    {
        var user = await _repository.UpdateAsync(userId, request);

        // Invalidate cache on update
        var cacheKey = $"user:{userId}";
        _cache.Remove(cacheKey);

        return user;
    }
}
```

## Cache-Aside Pattern

Lazy loading with cache-aside:

```csharp
public class CachedUserService : IUserService
{
    private readonly IUserRepository _repository;
    private readonly ICacheService _cache;
    private static readonly TimeSpan CacheDuration = TimeSpan.FromMinutes(5);

    public async Task<User?> GetUserByIdAsync(string userId)
    {
        return await GetOrSetAsync(
            key: $"user:{userId}",
            factory: () => _repository.GetByIdAsync(userId),
            expiration: CacheDuration);
    }

    public async Task<User[]> GetUsersAsync(int page, int limit)
    {
        return await GetOrSetAsync(
            key: $"users:page:{page}:limit:{limit}",
            factory: () => _repository.GetUsersAsync(page, limit),
            expiration: TimeSpan.FromMinutes(1));  // Shorter TTL for lists
    }

    private async Task<T?> GetOrSetAsync<T>(
        string key,
        Func<Task<T?>> factory,
        TimeSpan expiration)
    {
        // Try cache
        if (_cache.TryGet<T>(key, out var cachedValue))
        {
            return cachedValue;
        }

        // Cache miss - fetch from source
        var value = await factory();

        if (value != null)
        {
            _cache.Set(key, value, expiration);
        }

        return value;
    }
}
```

## Distributed Caching (Redis)

Use Redis for multi-instance caching:

```csharp
public class RedisCacheService : ICacheService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisCacheService> _logger;

    public RedisCacheService(
        IConnectionMultiplexer redis,
        ILogger<RedisCacheService> logger)
    {
        _redis = redis;
        _logger = logger;
    }

    public T? Get<T>(string key)
    {
        var db = _redis.GetDatabase();
        var value = db.StringGet(key);

        if (value.IsNullOrEmpty)
            return default;

        return JsonSerializer.Deserialize<T>(value!);
    }

    public void Set<T>(string key, T value, TimeSpan? expiration = null)
    {
        var db = _redis.GetDatabase();
        var serialized = JsonSerializer.Serialize(value);

        db.StringSet(key, serialized, expiration ?? TimeSpan.FromMinutes(5));

        _logger.LogDebug("Cached item in Redis with key: {Key}", key);
    }

    public void Remove(string key)
    {
        var db = _redis.GetDatabase();
        db.KeyDelete(key);

        _logger.LogDebug("Removed cached item from Redis with key: {Key}", key);
    }

    public bool TryGet<T>(string key, out T? value)
    {
        value = Get<T>(key);
        return value != null;
    }
}

// Register
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
{
    var configuration = ConfigurationOptions.Parse("localhost:6379");
    return ConnectionMultiplexer.Connect(configuration);
});

builder.Services.AddSingleton<ICacheService, RedisCacheService>();
```

AWS ElastiCache (Redis):

```hcl
# Terraform
resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "${var.service_name}-${var.environment}"
  engine               = "redis"
  node_type            = "cache.t3.micro"
  num_cache_nodes      = 1
  parameter_group_name = "default.redis7"
  port                 = 6379

  subnet_group_name = aws_elasticache_subnet_group.redis.name
  security_group_ids = [aws_security_group.redis.id]

  tags = {
    Environment = var.environment
    Service     = var.service_name
  }
}
```

## Response Caching

HTTP response caching:

```csharp
// Configure response caching
builder.Services.AddResponseCaching();

app.UseResponseCaching();

// Cache GET responses
app.MapGet("/users", async (IUserService userService) =>
{
    var users = await userService.GetUsersAsync();
    return Results.Ok(users);
})
.CacheOutput(policy => policy.Expire(TimeSpan.FromMinutes(5)));

// With cache variations
app.MapGet("/users/{id}", async (string id, IUserService userService) =>
{
    var user = await userService.GetUserByIdAsync(id);
    return user != null ? Results.Ok(user) : Results.NotFound();
})
.CacheOutput(policy => policy
    .Expire(TimeSpan.FromMinutes(5))
    .SetVaryByQuery("includeLoans", "includeDetails"));
```

Output caching (ASP.NET Core 7+):

```csharp
builder.Services.AddOutputCache(options =>
{
    // Default policy
    options.AddBasePolicy(builder => builder.Expire(TimeSpan.FromMinutes(1)));

    // Named policies
    options.AddPolicy("Users", builder =>
        builder.Expire(TimeSpan.FromMinutes(5)));

    options.AddPolicy("UserDetails", builder =>
        builder
            .Expire(TimeSpan.FromMinutes(10))
            .Tag("user-details")
            .SetVaryByQuery("includeLoans"));
});

app.UseOutputCache();

// Use output cache
app.MapGet("/users", GetUsers)
    .CacheOutput("Users");

app.MapGet("/users/{id}", GetUser)
    .CacheOutput("UserDetails");

// Invalidate by tag
app.MapPut("/users/{id}", async (
    string id,
    UpdateUserRequest request,
    IUserService userService,
    IOutputCacheStore cacheStore) =>
{
    var user = await userService.UpdateUserAsync(id, request);

    // Evict cached user details
    await cacheStore.EvictByTagAsync("user-details", default);

    return Results.Ok(user);
});
```

## Cache Invalidation

Invalidate on write:

```csharp
public class UserService : IUserService
{
    private readonly IUserRepository _repository;
    private readonly ICacheService _cache;

    public async Task<User> UpdateUserAsync(string userId, UpdateUserRequest request)
    {
        var user = await _repository.UpdateAsync(userId, request);

        // Invalidate related caches
        InvalidateUserCaches(userId);

        return user;
    }

    public async Task DeleteUserAsync(string userId)
    {
        await _repository.DeleteAsync(userId);

        // Invalidate all related caches
        InvalidateUserCaches(userId);
    }

    private void InvalidateUserCaches(string userId)
    {
        _cache.Remove($"user:{userId}");
        _cache.Remove($"user:{userId}:loans");
        _cache.Remove($"user:{userId}:details");

        // Invalidate list caches (could be more sophisticated)
        _cache.Remove("users:page:1:limit:20");
        _cache.Remove("users:page:1:limit:50");
    }
}
```

Time-based expiration:

```csharp
// Short TTL for frequently changing data
_cache.Set("active-users", users, TimeSpan.FromSeconds(30));

// Medium TTL for semi-static data
_cache.Set("user-settings", settings, TimeSpan.FromMinutes(5));

// Long TTL for static data
_cache.Set("static-content", content, TimeSpan.FromHours(1));

// Use sliding expiration for active data
var options = new MemoryCacheEntryOptions
{
    SlidingExpiration = TimeSpan.FromMinutes(5)  // Renew on access
};
_cache.Set("recent-activity", activity, options);
```

## Cache Keys

Structured cache keys:

```csharp
public static class CacheKeys
{
    private const string Prefix = "library";

    public static string User(string userId) =>
        $"{Prefix}:user:{userId}";

    public static string UserLoans(string userId) =>
        $"{Prefix}:user:{userId}:loans";

    public static string UsersList(int page, int limit) =>
        $"{Prefix}:users:page:{page}:limit:{limit}";

    public static string Book(string bookId) =>
        $"{Prefix}:book:{bookId}";

    public static string BooksByAuthor(string authorId) =>
        $"{Prefix}:books:author:{authorId}";

    // Pattern for wildcard deletion
    public static string UserPattern(string userId) =>
        $"{Prefix}:user:{userId}:*";
}

// Usage
var user = await GetOrSetAsync(
    CacheKeys.User(userId),
    () => _repository.GetByIdAsync(userId),
    TimeSpan.FromMinutes(5));
```

## Cache Stampede Prevention

Prevent thundering herd:

```csharp
public class StampedeProtectedCache
{
    private readonly ICacheService _cache;
    private readonly SemaphoreSlim _semaphore = new(1, 1);

    public async Task<T?> GetOrSetAsync<T>(
        string key,
        Func<Task<T?>> factory,
        TimeSpan expiration)
    {
        // Try cache first (no lock)
        if (_cache.TryGet<T>(key, out var cachedValue))
        {
            return cachedValue;
        }

        // Acquire lock before fetching
        await _semaphore.WaitAsync();
        try
        {
            // Double-check cache (another thread may have populated it)
            if (_cache.TryGet<T>(key, out cachedValue))
            {
                return cachedValue;
            }

            // Fetch from source
            var value = await factory();

            if (value != null)
            {
                _cache.Set(key, value, expiration);
            }

            return value;
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

## CDN Caching

CloudFront for static assets:

```hcl
# Terraform
resource "aws_cloudfront_distribution" "assets" {
  enabled = true
  comment = "${var.service_name} CDN"

  origin {
    domain_name = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id   = "S3-${aws_s3_bucket.assets.id}"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.assets.cloudfront_access_identity_path
    }
  }

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-${aws_s3_bucket.assets.id}"
    viewer_protocol_policy = "redirect-to-https"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 0
    default_ttl = 86400    # 1 day
    max_ttl     = 31536000  # 1 year
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```

Cache-Control headers:

```csharp
app.MapGet("/assets/{filename}", (string filename) =>
{
    var file = GetFile(filename);

    return Results.File(
        file,
        contentType: "image/png",
        fileDownloadName: filename,
        lastModified: DateTimeOffset.UtcNow,
        entityTag: new EntityTagHeaderValue($"\"{GenerateETag(file)}\""),
        enableRangeProcessing: true);
})
.WithMetadata(new ResponseCacheAttribute
{
    Duration = 86400,  // 1 day
    Location = ResponseCacheLocation.Any,
    VaryByHeader = "Accept-Encoding"
});
```

## Cache Monitoring

Monitor cache performance:

```csharp
public class MonitoredCacheService : ICacheService
{
    private readonly ICacheService _inner;
    private readonly ILogger<MonitoredCacheService> _logger;
    private long _hits;
    private long _misses;

    public T? Get<T>(string key)
    {
        var value = _inner.Get<T>(key);

        if (value != null)
        {
            Interlocked.Increment(ref _hits);
            _logger.LogDebug("Cache hit for key: {Key}", key);
        }
        else
        {
            Interlocked.Increment(ref _misses);
            _logger.LogDebug("Cache miss for key: {Key}", key);
        }

        return value;
    }

    public void Set<T>(string key, T value, TimeSpan? expiration = null)
    {
        _inner.Set(key, value, expiration);
        _logger.LogDebug("Cache set for key: {Key}, expiration: {Expiration}", key, expiration);
    }

    public (long Hits, long Misses, double HitRate) GetStatistics()
    {
        var hits = Interlocked.Read(ref _hits);
        var misses = Interlocked.Read(ref _misses);
        var total = hits + misses;
        var hitRate = total > 0 ? (double)hits / total : 0;

        return (hits, misses, hitRate);
    }
}

// Expose metrics endpoint
app.MapGet("/metrics/cache", (MonitoredCacheService cache) =>
{
    var (hits, misses, hitRate) = cache.GetStatistics();

    return Results.Ok(new
    {
        hits,
        misses,
        hitRate = $"{hitRate:P2}",
        total = hits + misses
    });
});
```

## Guidelines

**When to Cache:**
- Expensive database queries
- External API calls
- Computed results
- Frequently accessed data
- Rarely changing data

**When NOT to Cache:**
- User-specific sensitive data (without proper isolation)
- Rapidly changing data
- Large objects (> 1 MB)
- Data that must be real-time

**Cache Keys:**
- Use structured, hierarchical keys
- Include version in key for schema changes
- Avoid special characters
- Keep keys short but descriptive

**TTL Strategy:**
- Short TTL (seconds) for rapidly changing data
- Medium TTL (minutes) for semi-static data
- Long TTL (hours) for static data
- Use sliding expiration for active data

**Invalidation:**
- Invalidate on write operations
- Use cache tags for bulk invalidation
- Monitor stale data issues
- Implement cache warming for critical data

**Monitoring:**
- Track cache hit/miss ratio
- Monitor cache memory usage
- Alert on low hit rates
- Log cache operations

## Benefits

Performance. Reduce database load and response time.

Scalability. Handle more requests with same resources.

Cost. Lower infrastructure costs.

Availability. Serve cached data during outages.

## Related

- [performance-optimization.md](./performance-optimization.md) - General optimization
- [monitoring-observability.md](./monitoring-observability.md) - Performance monitoring
- [lambda-best-practices.md](../05-infrastructure/02-aws-lambda/lambda-best-practices.md) - Lambda caching
