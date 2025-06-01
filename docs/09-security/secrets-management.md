# Secrets Management

Secure secrets storage and access. Never commit secrets. Use AWS Secrets Manager or Parameter Store.

## Principle

Never hardcode secrets. Store securely. Rotate regularly. Audit access. Principle of least privilege.

## AWS Secrets Manager

Store secrets in AWS Secrets Manager:

```bash
# Create secret
aws secretsmanager create-secret \
  --name library-service/prod/database \
  --secret-string '{"username":"admin","password":"secure-password"}'

# Create API key secret
aws secretsmanager create-secret \
  --name library-service/prod/api-keys \
  --secret-string '{"external-api":"sk-1234567890"}'
```

Retrieve secrets in application:

```csharp
public interface ISecretsService
{
    Task<string> GetSecretAsync(string secretName);
    Task<T> GetSecretAsync<T>(string secretName) where T : class;
}

public class SecretsService : ISecretsService
{
    private readonly IAmazonSecretsManager _secretsManager;
    private readonly ILogger<SecretsService> _logger;
    private readonly IMemoryCache _cache;

    public SecretsService(
        IAmazonSecretsManager secretsManager,
        ILogger<SecretsService> logger,
        IMemoryCache cache)
    {
        _secretsManager = secretsManager;
        _logger = logger;
        _cache = cache;
    }

    public async Task<string> GetSecretAsync(string secretName)
    {
        // Check cache
        if (_cache.TryGetValue(secretName, out string? cachedSecret))
        {
            return cachedSecret!;
        }

        try
        {
            var request = new GetSecretValueRequest
            {
                SecretId = secretName
            };

            var response = await _secretsManager.GetSecretValueAsync(request);

            var secretValue = response.SecretString;

            // Cache for 5 minutes
            _cache.Set(secretName, secretValue, TimeSpan.FromMinutes(5));

            return secretValue;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to retrieve secret: {SecretName}", secretName);
            throw;
        }
    }

    public async Task<T> GetSecretAsync<T>(string secretName) where T : class
    {
        var secretJson = await GetSecretAsync(secretName);
        return JsonSerializer.Deserialize<T>(secretJson)
            ?? throw new InvalidOperationException($"Failed to deserialize secret: {secretName}");
    }
}

// Register
builder.Services.AddSingleton<IAmazonSecretsManager>(sp =>
    new AmazonSecretsManagerClient(RegionEndpoint.USEast1));

builder.Services.AddSingleton<ISecretsService, SecretsService>();
builder.Services.AddMemoryCache();
```

Usage:

```csharp
public class ExternalApiClient
{
    private readonly ISecretsService _secretsService;
    private readonly HttpClient _httpClient;

    public async Task<string> CallApiAsync()
    {
        // Get API key from secrets
        var apiKey = await _secretsService.GetSecretAsync("library-service/prod/api-key");

        _httpClient.DefaultRequestHeaders.Add("Authorization", $"Bearer {apiKey}");

        var response = await _httpClient.GetAsync("https://api.external.com/data");
        return await response.Content.ReadAsStringAsync();
    }
}
```

## AWS Systems Manager Parameter Store

Alternative to Secrets Manager:

```bash
# Create parameter
aws ssm put-parameter \
  --name /library-service/prod/jwt-secret \
  --type SecureString \
  --value "your-secret-key-at-least-32-characters-long"

# Create parameter with description
aws ssm put-parameter \
  --name /library-service/prod/database-connection \
  --type SecureString \
  --value "Server=db.example.com;Database=library;..." \
  --description "Production database connection string"
```

Retrieve parameters:

```csharp
public class ParameterStoreService
{
    private readonly IAmazonSimpleSystemsManagement _ssmClient;
    private readonly IMemoryCache _cache;

    public async Task<string> GetParameterAsync(string parameterName)
    {
        // Check cache
        if (_cache.TryGetValue(parameterName, out string? cachedValue))
        {
            return cachedValue!;
        }

        var request = new GetParameterRequest
        {
            Name = parameterName,
            WithDecryption = true
        };

        var response = await _ssmClient.GetParameterAsync(request);
        var value = response.Parameter.Value;

        // Cache for 5 minutes
        _cache.Set(parameterName, value, TimeSpan.FromMinutes(5));

        return value;
    }

    public async Task<Dictionary<string, string>> GetParametersByPathAsync(string path)
    {
        var request = new GetParametersByPathRequest
        {
            Path = path,
            WithDecryption = true,
            Recursive = true
        };

        var response = await _ssmClient.GetParametersByPathAsync(request);

        return response.Parameters.ToDictionary(
            p => p.Name,
            p => p.Value);
    }
}

// Register
builder.Services.AddSingleton<IAmazonSimpleSystemsManagement>(sp =>
    new AmazonSimpleSystemsManagementClient(RegionEndpoint.USEast1));

builder.Services.AddSingleton<ParameterStoreService>();
```

## Environment Variables

Configuration hierarchy:

```csharp
// appsettings.json (non-secrets only)
{
  "Database": {
    "TableName": "library-service-prod"
  },
  "Jwt": {
    "Issuer": "https://api.example.com",
    "Audience": "https://api.example.com",
    "ExpirationMinutes": 15
  }
}

// Environment variables (secrets)
// DATABASE_CONNECTION_STRING=...
// JWT_SECRET_KEY=...

// Configuration
var connectionString = builder.Configuration["DATABASE_CONNECTION_STRING"]
    ?? throw new InvalidOperationException("Database connection string not configured");

var jwtSecret = builder.Configuration["JWT_SECRET_KEY"]
    ?? throw new InvalidOperationException("JWT secret key not configured");
```

Lambda environment variables:

```hcl
# Terraform
resource "aws_lambda_function" "api" {
  function_name = "${var.service_name}-api-${var.environment}"

  environment {
    variables = {
      DYNAMODB_TABLE_NAME = aws_dynamodb_table.main.name
      ENVIRONMENT         = var.environment

      # Reference secrets from Parameter Store
      JWT_SECRET_ARN = aws_secretsmanager_secret.jwt_secret.arn
    }
  }
}

# Grant permission to read secret
resource "aws_iam_role_policy" "secrets_access" {
  name = "secrets-access"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["secretsmanager:GetSecretValue"]
      Resource = aws_secretsmanager_secret.jwt_secret.arn
    }]
  })
}
```

## User Secrets (Development)

Local development secrets:

```bash
# Initialize user secrets
dotnet user-secrets init --project src/LibraryService/LibraryService.Api

# Set secret
dotnet user-secrets set "Jwt:SecretKey" "dev-secret-key-at-least-32-characters" \
  --project src/LibraryService/LibraryService.Api

# Set connection string
dotnet user-secrets set "ConnectionStrings:Database" "Server=localhost;..." \
  --project src/LibraryService/LibraryService.Api

# List secrets
dotnet user-secrets list --project src/LibraryService/LibraryService.Api
```

Access in code:

```csharp
// Automatically loaded in Development environment
var jwtSecret = builder.Configuration["Jwt:SecretKey"];
```

## Secret Rotation

Implement secret rotation:

```csharp
public class SecretRotationService : BackgroundService
{
    private readonly ISecretsService _secretsService;
    private readonly ILogger<SecretRotationService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // Check if rotation needed
                var lastRotation = await GetLastRotationDateAsync();

                if (DateTime.UtcNow - lastRotation > TimeSpan.FromDays(90))
                {
                    await RotateSecretsAsync();
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error during secret rotation");
            }

            // Check daily
            await Task.Delay(TimeSpan.FromDays(1), stoppingToken);
        }
    }

    private async Task RotateSecretsAsync()
    {
        _logger.LogInformation("Rotating secrets");

        // Generate new secret
        var newSecret = GenerateSecureSecret();

        // Update in Secrets Manager
        await _secretsService.UpdateSecretAsync("library-service/prod/jwt-secret", newSecret);

        // Clear cache
        ClearSecretCache();

        _logger.LogInformation("Secrets rotated successfully");
    }

    private string GenerateSecureSecret()
    {
        var bytes = new byte[32];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(bytes);
        return Convert.ToBase64String(bytes);
    }
}
```

## Terraform Secrets

Manage secrets with Terraform:

```hcl
# Create secret
resource "aws_secretsmanager_secret" "jwt_secret" {
  name        = "${var.service_name}/${var.environment}/jwt-secret"
  description = "JWT signing secret for ${var.service_name}"

  tags = {
    Environment = var.environment
    Service     = var.service_name
  }
}

# Set initial value (manual or from file)
resource "aws_secretsmanager_secret_version" "jwt_secret" {
  secret_id     = aws_secretsmanager_secret.jwt_secret.id
  secret_string = var.jwt_secret_value  # From tfvars or external source
}

# Enable automatic rotation
resource "aws_secretsmanager_secret_rotation" "jwt_secret" {
  secret_id           = aws_secretsmanager_secret.jwt_secret.id
  rotation_lambda_arn = aws_lambda_function.secret_rotation.arn

  rotation_rules {
    automatically_after_days = 90
  }
}

# Parameter Store alternative
resource "aws_ssm_parameter" "database_password" {
  name        = "/${var.service_name}/${var.environment}/database-password"
  description = "Database password"
  type        = "SecureString"
  value       = var.database_password

  tags = {
    Environment = var.environment
    Service     = var.service_name
  }
}
```

Never commit secrets:

```hcl
# BAD - Never do this
resource "aws_secretsmanager_secret_version" "api_key" {
  secret_string = "sk-1234567890"  # NEVER hardcode!
}

# GOOD - Use variable
resource "aws_secretsmanager_secret_version" "api_key" {
  secret_string = var.api_key  # From tfvars or environment
}
```

## Secret Detection

Prevent secret commits:

```bash
# Install git-secrets
git clone https://github.com/awslabs/git-secrets
cd git-secrets
make install

# Install hooks
cd /path/to/repo
git secrets --install

# Add patterns
git secrets --add 'sk-[0-9a-zA-Z]{32,}'  # API keys
git secrets --add '[0-9]{12}'  # AWS account IDs
git secrets --add 'AKIA[0-9A-Z]{16}'  # AWS access keys

# Scan repository
git secrets --scan
```

Pre-commit hook:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

## Secret Scanning (CI/CD)

GitHub Actions:

```yaml
# .github/workflows/secret-scan.yml
name: Secret Scan

on: [push, pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: TruffleHog Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: main
          head: HEAD
```

## Guidelines

**Storage:**
- AWS Secrets Manager for secrets
- Parameter Store for configuration
- Never commit secrets to Git
- Use environment variables for Lambda

**Access:**
- Principle of least privilege
- IAM policies for secret access
- Audit secret access logs
- Rotate secrets regularly

**Development:**
- User secrets for local development
- Separate dev/prod secrets
- Never share secrets in chat/email
- Document secret requirements

**Rotation:**
- Rotate every 90 days
- Automate rotation process
- Test rotation procedure
- Monitor rotation failures

**Detection:**
- Pre-commit hooks
- CI/CD scanning
- Regular repository audits
- Revoke exposed secrets immediately

## Benefits

Secure. Secrets encrypted at rest.

Auditable. Track all secret access.

Automated. Rotation and deployment.

Compliant. Meets security standards.

## Related

- [authentication.md](./authentication.md) - JWT secrets
- [authorization.md](./authorization.md) - Permission management
- [owasp-security.md](./owasp-security.md) - Security best practices
- [terraform-best-practices.md](../05-infrastructure/01-terraform/terraform-best-practices.md) - Infrastructure security
