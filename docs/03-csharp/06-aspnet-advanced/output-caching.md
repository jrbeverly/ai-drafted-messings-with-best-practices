# Output Caching

Cache entire HTTP responses server-side to serve repeated requests in microseconds (.NET 7+).

## Why It Matters

- Bypasses the full pipeline for cached responses -- massive throughput increase
- Tag-based eviction keeps caches consistent on writes
- Built into ASP.NET Core, no external dependency required for single-server

## Key Recommendations

- **Basic setup**:
  ```csharp
  builder.Services.AddOutputCache();
  app.UseOutputCache();
  app.MapGet("/users", GetUsers).CacheOutput(p => p.Expire(TimeSpan.FromSeconds(60)));
  ```
- **Named policies** for reuse:
  ```csharp
  options.AddPolicy("Short", b => b.Expire(TimeSpan.FromSeconds(30)));
  options.AddPolicy("PerUser", b => b.Expire(TimeSpan.FromMinutes(5)).SetVaryByHeader("Authorization"));
  ```
- **Vary by** query params (`.SetVaryByQuery("page", "limit")`), headers, or custom values (tenant ID)
- **Tag-based eviction** -- invalidate related caches on writes:
  ```csharp
  // Tag on cache: .Tag("users")
  // Evict on write:
  await cacheStore.EvictByTagAsync("users", default);
  ```
- **Disable for specific endpoints**: `.CacheOutput(p => p.NoCache())` or `.DisableOutputCaching()`
- **Distributed cache**: implement `IOutputCacheStore` backed by Redis for multi-server deployments
- **Cache warming**: use a `BackgroundService` to pre-populate popular endpoints

## Cache Duration Guidelines

| Data type | Duration |
|-----------|----------|
| Dynamic / real-time | 30s-1m |
| Semi-static (products) | 5-10m |
| Static (categories) | 1h+ |

## Pitfalls to Avoid

- Caching POST/PUT/DELETE responses (only GET/HEAD should be cached)
- Forgetting to evict on writes (stale data served to users)
- Caching user-specific data without `VaryByHeader("Authorization")`
- Placing `UseOutputCache()` before `UseResponseCompression()` (compressed output not cached)
