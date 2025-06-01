# Health Checks

ASP.NET Core health endpoints for monitoring application health and dependency status.

## Why It Matters

- Load balancers and orchestrators use health checks to route traffic and restart unhealthy instances
- Quickly identify which dependency is failing without digging through logs
- Separate liveness (app is running) from readiness (app can serve traffic)

## Basic Setup

```csharp
builder.Services.AddHealthChecks()
    .AddCheck<DynamoDbHealthCheck>("dynamodb", tags: new[] { "ready" })
    .AddCheck<CacheHealthCheck>("cache", tags: new[] { "ready" });

app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // No checks -- returns 200 if app responds
});
```

## Custom Health Check

```csharp
public class DynamoDbHealthCheck(IAmazonDynamoDB dynamoDb) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken ct = default)
    {
        try
        {
            var resp = await dynamoDb.DescribeTableAsync(new DescribeTableRequest
            { TableName = "Users" }, ct);
            return resp.Table.TableStatus == TableStatus.ACTIVE
                ? HealthCheckResult.Healthy()
                : HealthCheckResult.Degraded($"Status: {resp.Table.TableStatus}");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("DynamoDB unreachable", ex);
        }
    }
}
```

## Liveness vs Readiness

| Probe | Purpose | Fails means |
|---|---|---|
| Liveness (`/health/live`) | Is the process alive? | Restart the instance |
| Readiness (`/health/ready`) | Are dependencies ready? | Remove from load balancer |

## Key Recommendations

- Set short timeouts (3-5 seconds) per check
- Use tags to separate liveness from readiness probes
- Return JSON with per-check detail for debugging (custom `ResponseWriter`)
- For Lambda, rely on CloudWatch metrics instead of health endpoints

## Pitfalls to Avoid

- Exposing connection strings or API keys in health check responses
- Checking business logic or data correctness (check connectivity only)
- Blocking on slow dependencies -- report Degraded instead of timing out the whole check
- Making liveness depend on external services (liveness should be self-contained)

## Related

- [httpclient-factory.md](./httpclient-factory.md) -- External API health checks
- [dependency-injection.md](./dependency-injection.md) -- Health check registration
