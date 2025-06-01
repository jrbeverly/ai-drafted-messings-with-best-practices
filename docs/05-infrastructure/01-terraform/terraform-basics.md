# Terraform Basics

Infrastructure as code with Terraform. Declarative configuration, version controlled, reproducible deployments.

## Principle

Define infrastructure in code. Version control everything. Separate environments. Use modules for reusability.

## Project Structure

Organize Terraform files by service and environment:

```
env/
├── library-service/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── terraform.tfvars
│       └── backend.tf
└── modules/
    ├── lambda-function/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── dynamodb-table/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

## Basic Configuration

Main configuration file:

```hcl
# env/library-service/dev/main.tf
terraform {
  required_version = ">= 1.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      Service     = var.service_name
      ManagedBy   = "Terraform"
    }
  }
}

# DynamoDB table
resource "aws_dynamodb_table" "main" {
  name         = "${var.service_name}-${var.environment}"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "PK"
  range_key    = "SK"

  attribute {
    name = "PK"
    type = "S"
  }

  attribute {
    name = "SK"
    type = "S"
  }

  attribute {
    name = "GSI1PK"
    type = "S"
  }

  attribute {
    name = "GSI1SK"
    type = "S"
  }

  global_secondary_index {
    name            = "GSI1"
    hash_key        = "GSI1PK"
    range_key       = "GSI1SK"
    projection_type = "ALL"
  }

  ttl {
    attribute_name = "ExpiresAt"
    enabled        = true
  }

  point_in_time_recovery {
    enabled = var.environment == "prod"
  }

  tags = {
    Name = "${var.service_name}-${var.environment}"
  }
}

# Lambda function
resource "aws_lambda_function" "api" {
  filename         = "${var.build_path}/api.zip"
  function_name    = "${var.service_name}-api-${var.environment}"
  role             = aws_iam_role.lambda.arn
  handler          = "bootstrap"
  source_code_hash = filebase64sha256("${var.build_path}/api.zip")
  runtime          = "provided.al2023"
  timeout          = 30
  memory_size      = 512

  environment {
    variables = {
      DYNAMODB_TABLE_NAME = aws_dynamodb_table.main.name
      ENVIRONMENT         = var.environment
      LOG_LEVEL           = var.log_level
    }
  }

  tracing_config {
    mode = "Active"
  }
}

# Lambda IAM role
resource "aws_iam_role" "lambda" {
  name = "${var.service_name}-lambda-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

# Lambda basic execution policy
resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# DynamoDB access policy
resource "aws_iam_role_policy" "dynamodb" {
  name = "dynamodb-access"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem",
          "dynamodb:Query",
          "dynamodb:Scan",
          "dynamodb:BatchGetItem",
          "dynamodb:BatchWriteItem",
          "dynamodb:TransactWriteItems",
          "dynamodb:TransactGetItems"
        ]
        Resource = [
          aws_dynamodb_table.main.arn,
          "${aws_dynamodb_table.main.arn}/index/*"
        ]
      }
    ]
  })
}
```

## Variables

Define input variables:

```hcl
# env/library-service/dev/variables.tf
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
}

variable "service_name" {
  description = "Service name"
  type        = string
  default     = "library-service"
}

variable "build_path" {
  description = "Path to build artifacts"
  type        = string
  default     = "../../build"
}

variable "log_level" {
  description = "Application log level"
  type        = string
  default     = "Information"
}

variable "enable_deletion_protection" {
  description = "Enable deletion protection on critical resources"
  type        = bool
  default     = false
}
```

Variable values:

```hcl
# env/library-service/dev/terraform.tfvars
environment                  = "dev"
aws_region                   = "us-east-1"
log_level                    = "Debug"
enable_deletion_protection   = false
```

Production variables:

```hcl
# env/library-service/prod/terraform.tfvars
environment                  = "prod"
aws_region                   = "us-east-1"
log_level                    = "Warning"
enable_deletion_protection   = true
```

## Outputs

Export resource attributes:

```hcl
# env/library-service/dev/outputs.tf
output "dynamodb_table_name" {
  description = "Name of the DynamoDB table"
  value       = aws_dynamodb_table.main.name
}

output "dynamodb_table_arn" {
  description = "ARN of the DynamoDB table"
  value       = aws_dynamodb_table.main.arn
}

output "lambda_function_name" {
  description = "Name of the Lambda function"
  value       = aws_lambda_function.api.function_name
}

output "lambda_function_arn" {
  description = "ARN of the Lambda function"
  value       = aws_lambda_function.api.arn
}

output "lambda_invoke_arn" {
  description = "Invoke ARN for API Gateway integration"
  value       = aws_lambda_function.api.invoke_arn
}
```

## Remote State

Store state in S3 with locking:

```hcl
# env/library-service/dev/backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "library-service/dev/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

Create state backend resources:

```hcl
# env/bootstrap/main.tf
# Run this once to create state backend

resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state"
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## Data Sources

Reference existing resources:

```hcl
# Get current AWS account ID
data "aws_caller_identity" "current" {}

# Get current region
data "aws_region" "current" {}

# Get VPC by tag
data "aws_vpc" "main" {
  tags = {
    Name = "main-vpc"
  }
}

# Use in resource
resource "aws_security_group" "lambda" {
  name   = "${var.service_name}-lambda-${var.environment}"
  vpc_id = data.aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## Locals

Computed values:

```hcl
locals {
  # Common tags
  common_tags = {
    Environment = var.environment
    Service     = var.service_name
    ManagedBy   = "Terraform"
    CostCenter  = "Engineering"
  }

  # Naming prefix
  name_prefix = "${var.service_name}-${var.environment}"

  # Is production
  is_production = var.environment == "prod"

  # Lambda configuration
  lambda_timeout = local.is_production ? 30 : 60
  lambda_memory  = local.is_production ? 1024 : 512
}

resource "aws_lambda_function" "api" {
  function_name = "${local.name_prefix}-api"
  timeout       = local.lambda_timeout
  memory_size   = local.lambda_memory

  tags = local.common_tags
}
```

## Conditional Resources

Create resources based on conditions:

```hcl
# CloudWatch alarm only in production
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  count = var.environment == "prod" ? 1 : 0

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
}

# Point-in-time recovery for production only
resource "aws_dynamodb_table" "main" {
  name = "${var.service_name}-${var.environment}"

  point_in_time_recovery {
    enabled = var.environment == "prod"
  }
}
```

## Lifecycle Rules

Control resource creation and destruction:

```hcl
resource "aws_dynamodb_table" "main" {
  name = "${var.service_name}-${var.environment}"

  lifecycle {
    # Prevent accidental deletion
    prevent_destroy = true

    # Ignore changes to read/write capacity
    ignore_changes = [
      read_capacity,
      write_capacity
    ]

    # Create new resource before destroying old
    create_before_destroy = true
  }
}
```

## Terraform Commands

Essential commands:

```bash
# Initialize Terraform (download providers)
terraform init

# Validate configuration syntax
terraform validate

# Format configuration files
terraform fmt -recursive

# Plan changes (preview)
terraform plan

# Apply changes
terraform apply

# Apply with auto-approve (CI/CD)
terraform apply -auto-approve

# Destroy infrastructure
terraform destroy

# Show current state
terraform show

# List resources in state
terraform state list

# Output values
terraform output

# Specific output
terraform output lambda_function_name

# Refresh state from actual infrastructure
terraform refresh

# Import existing resource
terraform import aws_dynamodb_table.main library-service-dev
```

## Workspace Management

Separate state per environment:

```bash
# List workspaces
terraform workspace list

# Create workspace
terraform workspace new dev

# Switch workspace
terraform workspace select dev

# Current workspace
terraform workspace show

# Delete workspace
terraform workspace delete dev
```

Using workspaces in configuration:

```hcl
locals {
  environment = terraform.workspace
}

resource "aws_lambda_function" "api" {
  function_name = "${var.service_name}-api-${local.environment}"
}
```

## Guidelines

**Project Structure:**
- One directory per environment (dev, staging, prod)
- Shared modules in separate directory
- Service-scoped infrastructure (one service per directory)
- Version control all Terraform files

**State Management:**
- Always use remote state (S3 + DynamoDB)
- Enable state locking
- Enable versioning on state bucket
- Encrypt state at rest

**Variables:**
- Define all variables in variables.tf
- Set values in terraform.tfvars
- Use environment-specific tfvars files
- Never commit secrets (use AWS Secrets Manager)

**Naming:**
- Consistent naming: `${service}-${resource}-${environment}`
- Use lowercase and hyphens
- Include environment in all resource names
- Use descriptive names

**Tags:**
- Tag all resources
- Use default_tags in provider
- Include: Environment, Service, ManagedBy
- Use tags for cost allocation

## Benefits

Reproducible. Infrastructure defined in code.

Version controlled. Track changes over time.

Declarative. Define desired state, Terraform handles how.

Safe. Plan changes before applying.

## Related

- [terraform-modules.md](./terraform-modules.md) - Reusable modules
- [terraform-best-practices.md](./terraform-best-practices.md) - Advanced patterns
- [aws-lambda-infrastructure.md](../02-aws-lambda/aws-lambda-infrastructure.md) - Lambda-specific patterns
