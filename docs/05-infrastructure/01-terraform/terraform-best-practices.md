# Terraform Best Practices

Advanced Terraform patterns for production infrastructure. Optimize for safety, cost, and maintainability.

## Principle

Plan before apply. Use workspaces or separate state files. Protect production. Automate everything.

## State Management

Never commit state files:

```gitignore
# .gitignore
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl
```

Use remote state with locking:

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-prod"
    key            = "library-service/prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"

    # Prevent accidental state modification
    skip_region_validation      = false
    skip_credentials_validation = false
    skip_metadata_api_check     = false
  }
}
```

State file encryption:

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
  }
}
```

## Resource Naming

Consistent naming convention:

```hcl
locals {
  # Format: {service}-{resource}-{environment}
  name_prefix = "${var.service_name}-${var.environment}"

  naming = {
    lambda    = "${local.name_prefix}-api"
    dynamodb  = "${local.name_prefix}-table"
    api       = "${local.name_prefix}-api"
    log_group = "/aws/${var.service_name}/${var.environment}"
  }
}

resource "aws_lambda_function" "api" {
  function_name = local.naming.lambda
  # ...
}

resource "aws_dynamodb_table" "main" {
  name = local.naming.dynamodb
  # ...
}
```

## Tagging Strategy

Comprehensive tagging:

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Service     = var.service_name
    ManagedBy   = "Terraform"
    CostCenter  = var.cost_center
    Owner       = var.owner_email
    Repository  = "github.com/org/repo"
  }
}

provider "aws" {
  region = var.aws_region

  # Apply tags to all resources
  default_tags {
    tags = local.common_tags
  }
}

# Resource-specific tags
resource "aws_lambda_function" "api" {
  function_name = local.naming.lambda

  tags = {
    Component = "API"
    Runtime   = "dotnet8"
  }
}
```

## Variable Validation

Validate inputs:

```hcl
variable "environment" {
  description = "Environment name"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod"
  }
}

variable "lambda_memory_size" {
  description = "Lambda memory in MB"
  type        = number

  validation {
    condition     = var.lambda_memory_size >= 128 && var.lambda_memory_size <= 10240
    error_message = "Lambda memory must be between 128 and 10240 MB"
  }
}

variable "aws_region" {
  description = "AWS region"
  type        = string

  validation {
    condition     = can(regex("^(us|eu|ap|sa|ca|me|af)-(north|south|east|west|central|northeast|southeast)-[1-3]$", var.aws_region))
    error_message = "Must be a valid AWS region"
  }
}
```

## Dynamic Blocks

Conditional resource configuration:

```hcl
resource "aws_lambda_function" "api" {
  function_name = local.naming.lambda

  # VPC config only if specified
  dynamic "vpc_config" {
    for_each = var.vpc_config != null ? [var.vpc_config] : []

    content {
      subnet_ids         = vpc_config.value.subnet_ids
      security_group_ids = vpc_config.value.security_group_ids
    }
  }

  # Dead letter config only in production
  dynamic "dead_letter_config" {
    for_each = var.environment == "prod" ? [1] : []

    content {
      target_arn = aws_sqs_queue.dlq[0].arn
    }
  }

  # Environment variables
  dynamic "environment" {
    for_each = length(var.environment_variables) > 0 ? [1] : []

    content {
      variables = var.environment_variables
    }
  }
}
```

## For Each Pattern

Create multiple similar resources:

```hcl
# Multiple Lambda functions
locals {
  functions = {
    api = {
      handler     = "Api::Api.LambdaEntryPoint::FunctionHandlerAsync"
      timeout     = 30
      memory_size = 512
    }
    worker = {
      handler     = "Worker::Worker.Function::Handler"
      timeout     = 300
      memory_size = 1024
    }
    scheduler = {
      handler     = "Scheduler::Scheduler.Function::Handler"
      timeout     = 60
      memory_size = 256
    }
  }
}

resource "aws_lambda_function" "functions" {
  for_each = local.functions

  function_name = "${local.name_prefix}-${each.key}"
  handler       = each.value.handler
  timeout       = each.value.timeout
  memory_size   = each.value.memory_size
  runtime       = "dotnet8"
  role          = aws_iam_role.lambda[each.key].arn

  filename         = "${var.build_path}/${each.key}.zip"
  source_code_hash = filebase64sha256("${var.build_path}/${each.key}.zip")
}

resource "aws_iam_role" "lambda" {
  for_each = local.functions

  name = "${local.name_prefix}-${each.key}-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}
```

## Depends On

Explicit dependencies:

```hcl
resource "aws_lambda_function" "api" {
  function_name = local.naming.lambda

  # Ensure log group exists before function
  depends_on = [aws_cloudwatch_log_group.lambda]
}

resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${local.naming.lambda}"
  retention_in_days = var.log_retention_days
}

# Ensure IAM policy is attached before using Lambda
resource "aws_lambda_function" "api" {
  depends_on = [
    aws_iam_role_policy_attachment.lambda_basic,
    aws_iam_role_policy.dynamodb
  ]
}
```

## Data Lookups

Use data sources instead of hardcoding:

```hcl
# Current account ID
data "aws_caller_identity" "current" {}

# Current region
data "aws_region" "current" {}

# Availability zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Latest AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Use in resources
resource "aws_iam_policy" "dynamodb" {
  name = "${local.name_prefix}-dynamodb-access"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = ["dynamodb:*"]
      Resource = [
        "arn:aws:dynamodb:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:table/${var.table_name}"
      ]
    }]
  })
}
```

## Conditional Resources

Environment-specific resources:

```hcl
# CloudWatch alarm only in production
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  count = var.environment == "prod" ? 1 : 0

  alarm_name          = "${local.name_prefix}-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Sum"
  threshold           = 10

  alarm_actions = [aws_sns_topic.alerts[0].arn]
}

resource "aws_sns_topic" "alerts" {
  count = var.environment == "prod" ? 1 : 0
  name  = "${local.name_prefix}-alerts"
}

# Backup only in production
resource "aws_backup_plan" "dynamodb" {
  count = var.environment == "prod" ? 1 : 0
  name  = "${local.name_prefix}-backup"

  rule {
    rule_name         = "daily"
    target_vault_name = aws_backup_vault.main[0].name
    schedule          = "cron(0 5 * * ? *)"
  }
}
```

## Import Existing Resources

Import manually created resources:

```bash
# Import DynamoDB table
terraform import aws_dynamodb_table.main library-service-prod

# Import Lambda function
terraform import aws_lambda_function.api library-service-api-prod

# Import IAM role
terraform import aws_iam_role.lambda library-service-lambda-prod-role

# Show what would be imported
terraform plan -generate-config-out=generated.tf
```

## Terraform Cloud / Workspaces

Separate environments with workspaces:

```hcl
# Use workspace name as environment
locals {
  environment = terraform.workspace
  is_prod     = local.environment == "prod"
}

resource "aws_lambda_function" "api" {
  function_name = "${var.service_name}-api-${local.environment}"
  memory_size   = local.is_prod ? 1024 : 512
  timeout       = local.is_prod ? 30 : 60
}
```

Workspace commands:

```bash
# Create workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Switch workspace
terraform workspace select prod

# List workspaces
terraform workspace list

# Apply to current workspace
terraform apply
```

## Cost Optimization

Use on-demand billing:

```hcl
# DynamoDB on-demand (pay per request)
resource "aws_dynamodb_table" "main" {
  name         = local.naming.dynamodb
  billing_mode = "PAY_PER_REQUEST"  # Instead of PROVISIONED
}

# Lambda with ARM architecture (cheaper)
resource "aws_lambda_function" "api" {
  function_name = local.naming.lambda
  architectures = ["arm64"]  # 20% cheaper than x86_64
}

# S3 Intelligent-Tiering
resource "aws_s3_bucket_intelligent_tiering_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id
  name   = "entire-bucket"

  tiering {
    access_tier = "ARCHIVE_ACCESS"
    days        = 90
  }
}
```

## Security Best Practices

Least privilege IAM policies:

```hcl
# Specific actions, not wildcards
resource "aws_iam_role_policy" "dynamodb" {
  name = "dynamodb-access"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query"
        # NOT "dynamodb:*"
      ]
      Resource = [
        aws_dynamodb_table.main.arn,
        "${aws_dynamodb_table.main.arn}/index/*"
      ]
      # NOT "Resource": "*"
    }]
  })
}
```

Prevent deletion of critical resources:

```hcl
resource "aws_dynamodb_table" "main" {
  name = local.naming.dynamodb

  lifecycle {
    prevent_destroy = true
  }

  deletion_protection_enabled = var.environment == "prod"
}
```

## Secrets Management

Never hardcode secrets:

```hcl
# BAD - hardcoded secret
resource "aws_lambda_function" "api" {
  environment {
    variables = {
      API_KEY = "sk-1234567890"  # NEVER DO THIS
    }
  }
}

# GOOD - use AWS Secrets Manager
data "aws_secretsmanager_secret_version" "api_key" {
  secret_id = "${var.service_name}/${var.environment}/api-key"
}

resource "aws_lambda_function" "api" {
  environment {
    variables = {
      API_KEY_SECRET_ARN = data.aws_secretsmanager_secret_version.api_key.arn
    }
  }
}

# Grant permission to read secret
resource "aws_iam_role_policy" "secrets" {
  name = "secrets-access"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["secretsmanager:GetSecretValue"]
      Resource = data.aws_secretsmanager_secret_version.api_key.arn
    }]
  })
}
```

## Error Handling

Handle errors gracefully:

```hcl
# Use try() for optional attributes
locals {
  vpc_id = try(data.aws_vpc.main[0].id, null)
}

# Use can() for validation
variable "cidr_block" {
  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block"
  }
}

# Use coalesce() for defaults
locals {
  log_retention = coalesce(var.log_retention_days, 7)
}
```

## Drift Detection

Detect manual changes:

```bash
# Check for drift
terraform plan -detailed-exitcode

# Exit code meanings:
# 0 = no changes
# 1 = error
# 2 = changes detected (drift)

# Refresh state from actual infrastructure
terraform refresh

# Show differences
terraform show
```

## CI/CD Pipeline

Automate Terraform in CI/CD:

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths:
      - 'env/**'
  push:
    branches:
      - main
    paths:
      - 'env/**'

jobs:
  terraform:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0

      - name: Terraform Format
        run: terraform fmt -check -recursive

      - name: Terraform Init
        run: terraform init
        working-directory: env/library-service/dev

      - name: Terraform Validate
        run: terraform validate
        working-directory: env/library-service/dev

      - name: Terraform Plan
        run: terraform plan -no-color
        working-directory: env/library-service/dev
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve
        working-directory: env/library-service/dev
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## Guidelines

**State Management:**
- Always use remote state with S3
- Enable state locking with DynamoDB
- Never commit state files
- Encrypt state at rest

**Security:**
- Use AWS Secrets Manager for secrets
- Least privilege IAM policies
- Enable deletion protection for production
- Encrypt data at rest and in transit

**Organization:**
- One directory per environment
- Use modules for reusable components
- Consistent naming conventions
- Comprehensive tagging

**Safety:**
- Always plan before apply
- Use lifecycle rules to prevent deletion
- Enable drift detection
- Review changes in pull requests

**Cost:**
- Use on-demand billing where possible
- Clean up unused resources
- Use tags for cost allocation
- Regular cost reviews

## Benefits

Safety. Plan changes before applying.

Automation. Infrastructure as code in CI/CD.

Reproducibility. Same configuration, same result.

Auditability. Version controlled, tracked changes.

## Related

- [terraform-basics.md](./terraform-basics.md) - Fundamentals
- [terraform-modules.md](./terraform-modules.md) - Reusable modules
- [aws-lambda-infrastructure.md](../02-aws-lambda/aws-lambda-infrastructure.md) - Lambda patterns
