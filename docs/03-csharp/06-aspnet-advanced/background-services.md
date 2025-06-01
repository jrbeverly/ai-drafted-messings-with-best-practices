# Background Services and Hosted Services

Run long-running tasks, scheduled jobs, and queue processors independently of HTTP requests.

## Why It Matters

- Decouple background work (email sending, cleanup) from request handling
- Graceful shutdown ensures in-progress work completes safely
- Health monitoring keeps background processors observable

## Key Patterns

| Need | Base Class | Notes |
|------|-----------|-------|
| Long-running loop | `BackgroundService` | Override `ExecuteAsync` |
| Startup/shutdown only | `IHostedService` | `StartAsync` / `StopAsync` |

## Key Recommendations

- **Create a scope for each unit of work** -- never inject scoped services via constructor:
  ```csharp
  using var scope = _serviceProvider.CreateScope();
  var svc = scope.ServiceProvider.GetRequiredService<IMyService>();
  ```
- **Use `PeriodicTimer`** for timed work (cleaner than `Task.Delay` loops):
  ```csharp
  using var timer = new PeriodicTimer(TimeSpan.FromMinutes(15));
  while (await timer.WaitForNextTickAsync(stoppingToken)) { /* work */ }
  ```
- **Use `Channel<T>`** for in-process queue processing between endpoints and background workers
- **Catch exceptions inside the loop** and retry with backoff -- an unhandled exception kills the service
- **Handle `OperationCanceledException`** from `stoppingToken` gracefully (expected on shutdown)
- **Limit parallelism** with `Parallel.ForEachAsync` and `MaxDegreeOfParallelism`
- **Register via** `builder.Services.AddHostedService<T>()`

## Health Checks

Track `LastSuccessfulRun` and `IsRunning` properties, then expose them through `IHealthCheck` to detect stalled services.

## Pitfalls to Avoid

- Injecting scoped services directly into the constructor (throws at runtime)
- Not catching exceptions -- one failure stops the entire service
- Forgetting `stoppingToken` in `Task.Delay` calls (shutdown hangs)
- Running blocking/synchronous code inside `ExecuteAsync`
