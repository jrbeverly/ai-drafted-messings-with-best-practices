# Application Lifetime

Manage startup, shutdown, and lifecycle events via `IHostApplicationLifetime`.

## Why It Matters

- Graceful shutdown prevents data loss and dropped connections
- Lifecycle hooks let you run initialization and cleanup logic
- Kubernetes and load balancers depend on proper shutdown signaling

## Lifecycle Event Order

```
App starts → IHostedService.StartAsync() → ApplicationStarted
→ App runs → Shutdown signal (SIGTERM/Ctrl+C) → ApplicationStopping
→ IHostedService.StopAsync() → Drain in-flight requests → ApplicationStopped → Exit
```

## Key Recommendations

- **Register shutdown callbacks** on `app.Lifetime.ApplicationStopping` for cleanup (flush logs, drain DB connections)
- **Set shutdown timeout** to allow in-flight requests to finish:
  ```csharp
  builder.Host.ConfigureHostOptions(o => o.ShutdownTimeout = TimeSpan.FromSeconds(30));
  ```
- **Use `BackgroundService`** with `CancellationToken` for long-running tasks -- check `stoppingToken.IsCancellationRequested` frequently
- **Fail fast on startup** -- call `app.Lifetime.StopApplication()` if initialization (migrations, seed data) fails
- **Bind `CancellationToken` in endpoints** to detect client disconnects automatically
- **Expose health probes** (`/health/ready`, `/health/live`) and toggle readiness after startup completes

## Kubernetes Integration

- Map `/health/ready` and `/health/live` to readiness and liveness probes
- Handle SIGTERM in `ApplicationStopping` to drain connections
- Consider a preStop hook with a short delay so the load balancer deregisters the pod first

## Pitfalls to Avoid

- Forgetting to pass `CancellationToken` through async call chains
- Setting shutdown timeout too low (requests get force-killed)
- Running expensive initialization inside `ApplicationStarted` callbacks instead of `IHostedService.StartAsync`
- Swallowing `OperationCanceledException` without logging it
