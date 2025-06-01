# AWS Lambda Best Practices

Serverless function optimization. Fast cold starts, efficient execution, cost-effective design.

FIXME: The block blocks aren't useful
FIXME: This can largely be replaced with some simple succinct bullet points
FIXME: This isn't really "best practices" more so "things you can do to improve lambda performance"

## Principle

Minimize cold starts. Right-size memory. Use Lambda layers. Handle failures gracefully.

## Function Configuration

Optimal settings:

```csharp
// C# Lambda with .NET 8
public class Function
{
    // Use dependency injection
    private readonly IUserService _userService;
    private readonly ILogger<Function> _logger;

    public Function(IUserService userService, ILogger<Function> logger)
    {
        _userService = userService;
        _logger = logger;
    }

    [LambdaSerializer(typeof(DefaultLambdaJsonSerializer))]
    public async Task<APIGatewayProxyResponse> FunctionHandler(
        APIGatewayProxyRequest request,
        ILambdaContext context)
    {
        try
        {
            _logger.LogInformation("Processing request: {RequestId}", context.RequestId);

            // Business logic
            var result = await _userService.GetUserAsync(userId);

            return new APIGatewayProxyResponse
            {
                StatusCode = 200,
                Body = JsonSerializer.Serialize(result),
                Headers = new Dictionary<string, string>
                {
                    { "Content-Type", "application/json" }
                }
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing request");

            return new APIGatewayProxyResponse
            {
                StatusCode = 500,
                Body = JsonSerializer.Serialize(new { error = "Internal server error" })
            };
        }
    }
}
```

Startup class:

```csharp
// Startup.cs - Lambda initialization
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Environment variables
        var tableName = Environment.GetEnvironmentVariable("DYNAMODB_TABLE_NAME");
        var environment = Environment.GetEnvironmentVariable("ENVIRONMENT");

        // DynamoDB
        services.AddSingleton<IAmazonDynamoDB>(sp =>
        {
            var config = new AmazonDynamoDBConfig
            {
                RegionEndpoint = RegionEndpoint.USEast1
            };
            return new AmazonDynamoDBClient(config);
        });

        // Configuration
        services.Configure<DatabaseOptions>(options =>
        {
            options.TableName = tableName ?? throw new InvalidOperationException("DYNAMODB_TABLE_NAME not set");
        });

        // Services
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IUserService, UserService>();

        // Logging
        services.AddLogging(logging =>
        {
            logging.ClearProviders();
            logging.AddConsole();
            logging.AddAWSProvider();
        });
    }
}
```

## Memory and Timeout

Right-size resources:

```hcl
resource "aws_lambda_function" "api" {
  function_name = "${var.service_name}-api-${var.environment}"

  # Memory (CPU scales with memory)
  # 128 MB = low CPU
  # 512 MB = moderate CPU
  # 1024 MB = high CPU
  memory_size = 512

  # Timeout
  # API: 30 seconds
  # Worker: 300 seconds (5 minutes)
  # Max: 900 seconds (15 minutes)
  timeout = 30

  # Architecture
  # arm64 is 20% cheaper than x86_64
  architectures = ["arm64"]

  # Ephemeral storage (default 512 MB)
  ephemeral_storage {
    size = 512  # Up to 10240 MB if needed
  }
}
```

Performance testing:

```bash
# Test different memory configurations
# Use AWS Lambda Power Tuning:
# https://github.com/alexcasalboni/aws-lambda-power-tuning

# Monitor duration and cost
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=my-function \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics Average
```

## Cold Start Optimization

Minimize initialization time:

```csharp
// GOOD - Initialize outside handler
public class Function
{
    // Static clients reused across invocations
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
        // Fast execution
        var result = await _userService.GetUserAsync(userId);
        return CreateResponse(result);
    }
}

// BAD - Initialize in handler
public class Function
{
    public async Task<APIGatewayProxyResponse> FunctionHandler(
        APIGatewayProxyRequest request,
        ILambdaContext context)
    {
        // This runs on EVERY invocation (slow!)
        var dynamoDb = new AmazonDynamoDBClient();
        var httpClient = new HttpClient();

        // ...
    }
}
```

Lambda SnapStart (.NET 8+):

```hcl
resource "aws_lambda_function" "api" {
  function_name = "${var.service_name}-api-${var.environment}"

  # Enable SnapStart for faster cold starts
  snap_start {
    apply_on = "PublishedVersions"
  }
}
```

## Environment Variables

Configuration via environment:

```hcl
resource "aws_lambda_function" "api" {
  function_name = "${var.service_name}-api-${var.environment}"

  environment {
    variables = {
      DYNAMODB_TABLE_NAME = aws_dynamodb_table.main.name
      ENVIRONMENT         = var.environment
      LOG_LEVEL           = var.log_level
      AWS_REGION          = var.aws_region

      # Feature flags
      ENABLE_CACHE = var.environment == "prod" ? "true" : "false"

      # API endpoints
      EXTERNAL_API_URL = var.external_api_url
    }
  }
}
```

Access in code:

```csharp
public class Function
{
    private readonly string _tableName;
    private readonly string _environment;
    private readonly LogLevel _logLevel;

    public Function()
    {
        _tableName = Environment.GetEnvironmentVariable("DYNAMODB_TABLE_NAME")
            ?? throw new InvalidOperationException("DYNAMODB_TABLE_NAME not set");

        _environment = Environment.GetEnvironmentVariable("ENVIRONMENT")
            ?? "dev";

        var logLevelStr = Environment.GetEnvironmentVariable("LOG_LEVEL") ?? "Information";
        _logLevel = Enum.Parse<LogLevel>(logLevelStr);
    }
}
```

## Error Handling

Handle failures gracefully:

```csharp
public async Task<APIGatewayProxyResponse> FunctionHandler(
    APIGatewayProxyRequest request,
    ILambdaContext context)
{
    try
    {
        // Validate input
        if (string.IsNullOrEmpty(request.Body))
        {
            return new APIGatewayProxyResponse
            {
                StatusCode = 400,
                Body = JsonSerializer.Serialize(new { error = "Request body is required" })
            };
        }

        // Parse input
        var input = JsonSerializer.Deserialize<CreateUserRequest>(request.Body);

        // Business logic
        var user = await _userService.CreateUserAsync(input);

        return new APIGatewayProxyResponse
        {
            StatusCode = 201,
            Body = JsonSerializer.Serialize(user),
            Headers = new Dictionary<string, string>
            {
                { "Content-Type", "application/json" },
                { "Location", $"/users/{user.Id}" }
            }
        };
    }
    catch (ValidationException ex)
    {
        _logger.LogWarning(ex, "Validation error");
        return new APIGatewayProxyResponse
        {
            StatusCode = 400,
            Body = JsonSerializer.Serialize(new { error = ex.Message, errors = ex.Errors })
        };
    }
    catch (NotFoundException ex)
    {
        _logger.LogWarning(ex, "Resource not found");
        return new APIGatewayProxyResponse
        {
            StatusCode = 404,
            Body = JsonSerializer.Serialize(new { error = ex.Message })
        };
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Unhandled exception");
        return new APIGatewayProxyResponse
        {
            StatusCode = 500,
            Body = JsonSerializer.Serialize(new { error = "Internal server error" })
        };
    }
}
```

## Dead Letter Queue

Handle async failures:

```hcl
resource "aws_sqs_queue" "dlq" {
  name                      = "${var.service_name}-dlq-${var.environment}"
  message_retention_seconds = 1209600  # 14 days
}

resource "aws_lambda_function" "worker" {
  function_name = "${var.service_name}-worker-${var.environment}"

  dead_letter_config {
    target_arn = aws_sqs_queue.dlq.arn
  }
}

resource "aws_iam_role_policy" "dlq" {
  name = "dlq-access"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["sqs:SendMessage"]
      Resource = aws_sqs_queue.dlq.arn
    }]
  })
}
```

## Retry Configuration

Configure retry behavior:

```hcl
# Event source mapping (e.g., SQS, DynamoDB Streams)
resource "aws_lambda_event_source_mapping" "sqs" {
  event_source_arn = aws_sqs_queue.input.arn
  function_name    = aws_lambda_function.worker.arn

  # Batch size
  batch_size = 10

  # Maximum batching window
  maximum_batching_window_in_seconds = 5

  # Retry attempts
  maximum_retry_attempts = 3

  # Partial batch failures
  function_response_types = ["ReportBatchItemFailures"]
}
```

Implement partial batch processing:

```csharp
public class Function
{
    public async Task<SQSBatchResponse> FunctionHandler(
        SQSEvent sqsEvent,
        ILambdaContext context)
    {
        var batchItemFailures = new List<SQSBatchResponse.BatchItemFailure>();

        foreach (var record in sqsEvent.Records)
        {
            try
            {
                await ProcessMessageAsync(record);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process message: {MessageId}", record.MessageId);

                // Add to failures (will be retried)
                batchItemFailures.Add(new SQSBatchResponse.BatchItemFailure
                {
                    ItemIdentifier = record.MessageId
                });
            }
        }

        return new SQSBatchResponse
        {
            BatchItemFailures = batchItemFailures
        };
    }
}
```

## Logging

Structured logging:

```csharp
public class Function
{
    private readonly ILogger<Function> _logger;

    public async Task<APIGatewayProxyResponse> FunctionHandler(
        APIGatewayProxyRequest request,
        ILambdaContext context)
    {
        // Structured logging with properties
        _logger.LogInformation(
            "Processing request for user {UserId} from IP {IpAddress}",
            userId,
            request.RequestContext.Identity.SourceIp);

        // Log execution time
        var stopwatch = Stopwatch.StartNew();
        var result = await _userService.GetUserAsync(userId);
        stopwatch.Stop();

        _logger.LogInformation(
            "Request completed in {ElapsedMs}ms",
            stopwatch.ElapsedMilliseconds);

        return CreateResponse(result);
    }
}
```

CloudWatch Insights queries:

```
# Find errors
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100

# Average duration
fields @duration
| stats avg(@duration) as avg_duration by bin(5m)

# Cold starts
fields @message
| filter @message like /INIT_START/
| stats count() as cold_starts by bin(1h)
```

## Monitoring

CloudWatch alarms:

```hcl
resource "aws_cloudwatch_metric_alarm" "errors" {
  alarm_name          = "${var.service_name}-lambda-errors-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Sum"
  threshold           = 10

  dimensions = {
    FunctionName = aws_lambda_function.api.function_name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "duration" {
  alarm_name          = "${var.service_name}-lambda-duration-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Duration"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Average"
  threshold           = 5000  # 5 seconds

  dimensions = {
    FunctionName = aws_lambda_function.api.function_name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "throttles" {
  alarm_name          = "${var.service_name}-lambda-throttles-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Throttles"
  namespace           = "AWS/Lambda"
  period              = 60
  statistic           = "Sum"
  threshold           = 0

  dimensions = {
    FunctionName = aws_lambda_function.api.function_name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

## Reserved Concurrency

Control concurrent executions:

```hcl
resource "aws_lambda_function" "api" {
  function_name = "${var.service_name}-api-${var.environment}"

  # Limit concurrent executions
  # Prevents runaway costs
  # Protects downstream services
  reserved_concurrent_executions = var.environment == "prod" ? 100 : 10
}
```

## Provisioned Concurrency

Eliminate cold starts for critical functions:

```hcl
resource "aws_lambda_function" "api" {
  function_name = "${var.service_name}-api-${var.environment}"
  publish       = true  # Required for provisioned concurrency
}

resource "aws_lambda_provisioned_concurrency_config" "api" {
  count = var.environment == "prod" ? 1 : 0

  function_name                     = aws_lambda_function.api.function_name
  provisioned_concurrent_executions = 5
  qualifier                         = aws_lambda_function.api.version
}
```

## Guidelines

**Function Design:**

- One function per endpoint or task
- Keep functions small and focused
- Initialize clients outside handler
- Use dependency injection

**Performance:**

- Right-size memory (test with Power Tuning)
- Use ARM architecture for 20% cost savings
- Enable SnapStart for faster cold starts
- Minimize package size

**Error Handling:**

- Handle all exceptions
- Use dead letter queues for async
- Implement partial batch processing
- Return appropriate HTTP status codes

**Monitoring:**

- Structured logging with context
- CloudWatch alarms for errors, duration, throttles
- Use X-Ray for distributed tracing
- Monitor cold start frequency

**Cost:**

- Set reserved concurrency to prevent runaway costs
- Use provisioned concurrency only when needed
- Monitor costs with CloudWatch metrics
- Clean up old versions

## Benefits

Serverless. No servers to manage.

Scalable. Automatic scaling.

Cost-effective. Pay per execution.

Fast. Millisecond response times.

## Related

- [lambda-infrastructure.md](./lambda-infrastructure.md) - Terraform patterns
- [lambda-dotnet-deployment.md](./lambda-dotnet-deployment.md) - .NET deployment
- [terraform-basics.md](../01-terraform/terraform-basics.md) - Infrastructure basics
