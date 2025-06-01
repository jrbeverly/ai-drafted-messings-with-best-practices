# Monitoring and Observability

Application monitoring, logging, metrics, and distributed tracing. Understand system behavior.

## Principle

Three pillars: logs, metrics, traces. Monitor what matters. Alert on actionable issues. Observability over monitoring.

## Structured Logging (Serilog)

Configure Serilog:

```csharp
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// Configure Serilog
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithProperty("Application", "LibraryService")
    .Enrich.WithProperty("Environment", builder.Environment.EnvironmentName)
    .WriteTo.Console(new JsonFormatter())
    .WriteTo.File(
        path: "logs/app-.log",
        rollingInterval: RollingInterval.Day,
        outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}")
    .CreateLogger();

builder.Host.UseSerilog();

var app = builder.Build();

try
{
    Log.Information("Starting application");
    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}
```

Structured logging with properties:

```csharp
public class UserService
{
    private readonly ILogger<UserService> _logger;

    public async Task<User> CreateUserAsync(CreateUserRequest request)
    {
        _logger.LogInformation(
            "Creating user with email {Email}",
            request.Email);

        var user = await _repository.CreateAsync(request);

        _logger.LogInformation(
            "User created successfully. UserId: {UserId}, Email: {Email}",
            user.Id,
            user.Email);

        return user;
    }

    public async Task<User?> GetUserAsync(string userId)
    {
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["UserId"] = userId,
            ["Operation"] = "GetUser"
        }))
        {
            _logger.LogDebug("Fetching user from repository");

            var user = await _repository.GetByIdAsync(userId);

            if (user == null)
            {
                _logger.LogWarning("User not found");
                return null;
            }

            _logger.LogDebug("User retrieved successfully");
            return user;
        }
    }
}
```

## CloudWatch Logs

Send logs to CloudWatch:

```csharp
Log.Logger = new LoggerConfiguration()
    .WriteTo.AmazonCloudWatch(
        logGroup: "/aws/lambda/library-service-api-prod",
        logStreamPrefix: DateTimeOffset.UtcNow.ToString("yyyyMMdd"),
        cloudWatchClient: new AmazonCloudWatchLogsClient())
    .CreateLogger();
```

Query logs with CloudWatch Insights:

```
# Find errors in last hour
fields @timestamp, @message, @logStream
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100

# Count errors by endpoint
fields @timestamp, httpMethod, path
| filter level = "Error"
| stats count() by path
| sort count desc

# Average response time by endpoint
fields @timestamp, duration, path
| filter @message like /Request completed/
| stats avg(duration) as avg_duration by path
| sort avg_duration desc

# Find slow requests (> 1 second)
fields @timestamp, duration, path, userId
| filter duration > 1000
| sort duration desc
| limit 50
```

## Metrics with CloudWatch

Custom metrics:

```csharp
public class MetricsService
{
    private readonly IAmazonCloudWatch _cloudWatch;
    private readonly string _namespace;

    public MetricsService(IAmazonCloudWatch cloudWatch, string @namespace)
    {
        _cloudWatch = cloudWatch;
        _namespace = @namespace;
    }

    public async Task RecordMetricAsync(
        string metricName,
        double value,
        StandardUnit unit,
        Dictionary<string, string>? dimensions = null)
    {
        var request = new PutMetricDataRequest
        {
            Namespace = _namespace,
            MetricData = new List<MetricDatum>
            {
                new MetricDatum
                {
                    MetricName = metricName,
                    Value = value,
                    Unit = unit,
                    TimestampUtc = DateTime.UtcNow,
                    Dimensions = dimensions?.Select(kvp => new Dimension
                    {
                        Name = kvp.Key,
                        Value = kvp.Value
                    }).ToList()
                }
            }
        };

        await _cloudWatch.PutMetricDataAsync(request);
    }

    public async Task IncrementCounterAsync(string metricName)
    {
        await RecordMetricAsync(metricName, 1, StandardUnit.Count);
    }
}

// Usage
public class UserService
{
    private readonly MetricsService _metrics;

    public async Task<User> CreateUserAsync(CreateUserRequest request)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            var user = await _repository.CreateAsync(request);

            await _metrics.IncrementCounterAsync("UserCreated");

            return user;
        }
        catch (Exception)
        {
            await _metrics.IncrementCounterAsync("UserCreationFailed");
            throw;
        }
        finally
        {
            stopwatch.Stop();

            await _metrics.RecordMetricAsync(
                "UserCreationDuration",
                stopwatch.ElapsedMilliseconds,
                StandardUnit.Milliseconds);
        }
    }
}
```

Application metrics:

```csharp
public class ApplicationMetrics
{
    private readonly MetricsService _metrics;

    public async Task RecordRequestAsync(
        string method,
        string path,
        int statusCode,
        long durationMs)
    {
        var dimensions = new Dictionary<string, string>
        {
            ["Method"] = method,
            ["Path"] = path,
            ["StatusCode"] = statusCode.ToString()
        };

        // Count requests
        await _metrics.RecordMetricAsync(
            "RequestCount",
            1,
            StandardUnit.Count,
            dimensions);

        // Record duration
        await _metrics.RecordMetricAsync(
            "RequestDuration",
            durationMs,
            StandardUnit.Milliseconds,
            dimensions);
    }

    public async Task RecordCacheHitAsync(bool hit)
    {
        await _metrics.IncrementCounterAsync(hit ? "CacheHit" : "CacheMiss");
    }

    public async Task RecordDatabaseQueryAsync(long durationMs)
    {
        await _metrics.RecordMetricAsync(
            "DatabaseQueryDuration",
            durationMs,
            StandardUnit.Milliseconds);
    }
}
```

## Distributed Tracing (AWS X-Ray)

Enable X-Ray tracing:

```csharp
builder.Services.AddAWSService<IAmazonXRay>();

app.UseXRay("LibraryService");

// Trace custom segments
public class UserService
{
    public async Task<User> GetUserAsync(string userId)
    {
        using (var segment = AWSXRayRecorder.Instance.BeginSubsegment("GetUser"))
        {
            segment.AddAnnotation("UserId", userId);
            segment.AddMetadata("Operation", "GetUser");

            try
            {
                var user = await _repository.GetByIdAsync(userId);

                segment.AddMetadata("UserFound", user != null);

                return user;
            }
            catch (Exception ex)
            {
                segment.AddException(ex);
                throw;
            }
        }
    }
}
```

Trace HTTP calls:

```csharp
builder.Services.AddHttpClient("ExternalApi")
    .AddHttpMessageHandler<HttpTracingHandler>();

public class HttpTracingHandler : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        using (var segment = AWSXRayRecorder.Instance.BeginSubsegment("HttpCall"))
        {
            segment.AddAnnotation("Url", request.RequestUri?.ToString() ?? "");
            segment.AddAnnotation("Method", request.Method.ToString());

            var stopwatch = Stopwatch.StartNew();

            try
            {
                var response = await base.SendAsync(request, cancellationToken);

                stopwatch.Stop();

                segment.AddMetadata("StatusCode", (int)response.StatusCode);
                segment.AddMetadata("Duration", stopwatch.ElapsedMilliseconds);

                return response;
            }
            catch (Exception ex)
            {
                segment.AddException(ex);
                throw;
            }
        }
    }
}
```

## Health Checks

Configure health checks:

```csharp
builder.Services.AddHealthChecks()
    .AddCheck<DynamoDbHealthCheck>("dynamodb")
    .AddCheck<ExternalApiHealthCheck>("external-api")
    .AddCheck<CacheHealthCheck>("cache");

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";

        var response = new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description,
                duration = e.Value.Duration.TotalMilliseconds
            }),
            totalDuration = report.TotalDuration.TotalMilliseconds
        };

        await context.Response.WriteAsJsonAsync(response);
    }
});

// DynamoDB health check
public class DynamoDbHealthCheck : IHealthCheck
{
    private readonly IAmazonDynamoDB _dynamoDb;
    private readonly string _tableName;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var request = new DescribeTableRequest { TableName = _tableName };
            var response = await _dynamoDb.DescribeTableAsync(request, cancellationToken);

            if (response.Table.TableStatus == TableStatus.ACTIVE)
            {
                return HealthCheckResult.Healthy("DynamoDB table is active");
            }

            return HealthCheckResult.Degraded($"DynamoDB table status: {response.Table.TableStatus}");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("DynamoDB is unavailable", ex);
        }
    }
}
```

## Alarms and Alerts

CloudWatch alarms:

```hcl
# Terraform
resource "aws_cloudwatch_metric_alarm" "high_error_rate" {
  alarm_name          = "${var.service_name}-high-error-rate-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "Alert when error rate is high"
  treat_missing_data  = "notBreaching"

  dimensions = {
    FunctionName = aws_lambda_function.api.function_name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "high_latency" {
  alarm_name          = "${var.service_name}-high-latency-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Duration"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Average"
  threshold           = 5000  # 5 seconds
  alarm_description   = "Alert when average latency is high"

  dimensions = {
    FunctionName = aws_lambda_function.api.function_name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

resource "aws_sns_topic" "alerts" {
  name = "${var.service_name}-alerts-${var.environment}"
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}
```

## Performance Monitoring

Request timing middleware:

```csharp
public class PerformanceMonitoringMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PerformanceMonitoringMiddleware> _logger;
    private readonly MetricsService _metrics;

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        var requestId = context.TraceIdentifier;

        try
        {
            await _next(context);

            stopwatch.Stop();

            _logger.LogInformation(
                "Request completed: {Method} {Path} - Status: {StatusCode} - Duration: {Duration}ms - RequestId: {RequestId}",
                context.Request.Method,
                context.Request.Path,
                context.Response.StatusCode,
                stopwatch.ElapsedMilliseconds,
                requestId);

            await _metrics.RecordRequestAsync(
                context.Request.Method,
                context.Request.Path,
                context.Response.StatusCode,
                stopwatch.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            stopwatch.Stop();

            _logger.LogError(ex,
                "Request failed: {Method} {Path} - Duration: {Duration}ms - RequestId: {RequestId}",
                context.Request.Method,
                context.Request.Path,
                stopwatch.ElapsedMilliseconds,
                requestId);

            await _metrics.IncrementCounterAsync("RequestError");

            throw;
        }
    }
}

app.UseMiddleware<PerformanceMonitoringMiddleware>();
```

## Dashboards

CloudWatch Dashboard:

```hcl
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "${var.service_name}-${var.environment}"

  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          metrics = [
            ["AWS/Lambda", "Invocations", { stat = "Sum", label = "Invocations" }],
            [".", "Errors", { stat = "Sum", label = "Errors" }],
            [".", "Throttles", { stat = "Sum", label = "Throttles" }]
          ]
          period = 300
          stat   = "Sum"
          region = var.aws_region
          title  = "Lambda Metrics"
        }
      },
      {
        type = "metric"
        properties = {
          metrics = [
            ["AWS/Lambda", "Duration", { stat = "Average" }],
            ["...", { stat = "Maximum" }],
            ["...", { stat = "p99" }]
          ]
          period = 300
          region = var.aws_region
          title  = "Latency"
        }
      },
      {
        type = "log"
        properties = {
          query   = "fields @timestamp, @message | filter level = \"Error\" | sort @timestamp desc | limit 20"
          region  = var.aws_region
          title   = "Recent Errors"
        }
      }
    ]
  })
}
```

## Error Tracking

Centralized error logging:

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;
    private readonly MetricsService _metrics;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var requestId = context.TraceIdentifier;

        _logger.LogError(exception,
            "Unhandled exception occurred. RequestId: {RequestId}, Path: {Path}, Method: {Method}, UserId: {UserId}",
            requestId,
            context.Request.Path,
            context.Request.Method,
            context.User.FindFirst("sub")?.Value ?? "anonymous");

        // Record error metric
        await _metrics.RecordMetricAsync(
            "UnhandledException",
            1,
            StandardUnit.Count,
            new Dictionary<string, string>
            {
                ["ExceptionType"] = exception.GetType().Name,
                ["Path"] = context.Request.Path
            });

        // Return error response
        context.Response.StatusCode = 500;
        await context.Response.WriteAsJsonAsync(new
        {
            error = "Internal server error",
            requestId
        }, cancellationToken);

        return true;
    }
}
```

## Guidelines

**Logging:**
- Use structured logging (JSON format)
- Include context (requestId, userId, etc.)
- Log levels: Debug < Info < Warning < Error < Fatal
- Don't log sensitive data (passwords, tokens)
- Log at boundaries (API calls, database queries)

**Metrics:**
- Track what matters (error rate, latency, throughput)
- Use dimensions for filtering (endpoint, user type)
- Set appropriate units (milliseconds, count, bytes)
- Aggregate metrics (average, sum, percentiles)

**Tracing:**
- Trace across service boundaries
- Include meaningful annotations
- Capture errors and exceptions
- Sample intelligently (100% for errors, 10% for success)

**Alerts:**
- Alert on actionable issues
- Avoid alert fatigue
- Include context in alerts
- Test alert delivery

**Dashboards:**
- Focus on key metrics
- Use appropriate visualizations
- Group related metrics
- Update regularly

## Benefits

Visibility. Understand system behavior.

Debugging. Quickly identify issues.

Performance. Track and optimize bottlenecks.

Reliability. Detect and resolve problems proactively.

## Related

- [caching-strategies.md](./caching-strategies.md) - Cache monitoring
- [performance-optimization.md](./performance-optimization.md) - Optimization techniques
- [lambda-best-practices.md](../05-infrastructure/02-aws-lambda/lambda-best-practices.md) - Lambda monitoring
