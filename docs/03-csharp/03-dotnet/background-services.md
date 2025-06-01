# Background Services

`IHostedService` and `BackgroundService` for long-running tasks that run independently of HTTP requests.

## Why It Matters

- Scheduled work (cleanup, metrics, queue processing) without external schedulers
- Lifecycle managed by the host -- automatic start, graceful shutdown
- Integrates with DI, configuration, and logging out of the box

## BackgroundService (Continuous Work)

```csharp
public class DataCleanupService : BackgroundService
{
    private readonly ILogger<DataCleanupService> _logger;
    private readonly IServiceScopeFactory _scopeFactory;

    public DataCleanupService(ILogger<DataCleanupService> logger, IServiceScopeFactory scopeFactory)
    { _logger = logger; _scopeFactory = scopeFactory; }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromHours(1));
        while (!stoppingToken.IsCancellationRequested
               && await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                using var scope = _scopeFactory.CreateScope();
                var repo = scope.ServiceProvider.GetRequiredService<IDataRepository>();
                await repo.DeleteOldRecordsAsync(stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "Cleanup failed");
            }
        }
    }
}

builder.Services.AddHostedService<DataCleanupService>();
```

## IHostedService (Startup/Shutdown Only)

```csharp
public class CacheWarmerService : IHostedService
{
    public async Task StartAsync(CancellationToken ct) => await _cache.WarmAsync(ct);
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

## Key Recommendations

- **Always use `IServiceScopeFactory`** to resolve scoped services -- background services are singletons
- **Use `PeriodicTimer`** over `Task.Delay` loops for cleaner interval scheduling
- **Pass `CancellationToken` through** every async call for graceful shutdown
- **Catch exceptions inside the loop** -- an unhandled exception kills the host
- **Implement exponential backoff** on consecutive failures

## When to Use Which

| Pattern | Use For |
|---|---|
| `BackgroundService` | Continuous/periodic work (queues, cleanup, metrics) |
| `IHostedService` | One-time startup init, cache warming, graceful shutdown |

## Pitfalls to Avoid

- Injecting scoped services directly into the constructor (use `IServiceScopeFactory`)
- Forgetting to handle `OperationCanceledException` during shutdown
- Running background work in Lambda (use EventBridge scheduled rules instead)
- Swallowing all exceptions silently -- log and consider a failure threshold

## Related

- [dependency-injection.md](./dependency-injection.md) -- IServiceScopeFactory usage
- [dependency-injection-lifetimes.md](./dependency-injection-lifetimes.md) -- Scoped services in singletons
