# Health Checks

Monitor application and dependency health for Kubernetes probes, load balancers, and alerting.

## Why It Matters

- Kubernetes uses health probes to restart crashed pods and remove unhealthy ones from traffic
- Early detection of degraded dependencies prevents cascading failures
- Structured health responses simplify debugging in production

## Key Recommendations

- **Separate liveness from readiness** with tags:
  ```csharp
  builder.Services.AddHealthChecks()
      .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { "live" })
      .AddCheck<DynamoDbHealthCheck>("dynamodb", tags: new[] { "ready" });

  app.MapHealthChecks("/health/live", new() { Predicate = c => c.Tags.Contains("live") });
  app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });
  ```
- **One check per dependency** -- keep each check focused and fast (< 1 second)
- **Use three status levels**: Healthy (serve traffic), Degraded (working but slow), Unhealthy (remove from LB)
- **Add a startup health check** with a `volatile bool` flag set after initialization completes
- **Return JSON details** via a custom `ResponseWriter` for debugging (status, duration, per-check data)
- **Publish metrics** to CloudWatch or similar with `IHealthCheckPublisher` on a periodic interval

## Kubernetes Probe Config

```yaml
livenessProbe:
  httpGet: { path: /health/live, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet: { path: /health/ready, port: 8080 }
  periodSeconds: 5
  failureThreshold: 3
startupProbe:
  httpGet: { path: /health/ready, port: 8080 }
  periodSeconds: 5
  failureThreshold: 30
```

## Pitfalls to Avoid

- Slow health checks that cause probe timeouts and unnecessary restarts
- Combining liveness and readiness into a single endpoint (causes restarts when only a dependency is down)
- Health checks with side effects (writes, cache mutations)
- Not setting `failureThreshold` -- one transient blip triggers a restart
