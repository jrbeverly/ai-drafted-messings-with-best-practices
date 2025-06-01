# Terraform Modules

Reusable infrastructure components. DRY principle for Terraform. Encapsulate complex logic.

## Principle

Extract common patterns into modules. One module per logical component. Versioned and tested independently.

## Module Structure

Standard module layout:

```
modules/
└── lambda-api/
    ├── main.tf           # Primary resources
    ├── variables.tf      # Input variables
    ├── outputs.tf        # Output values
    ├── versions.tf       # Provider requirements
    ├── README.md         # Documentation
    └── examples/         # Usage examples
        └── basic/
            ├── main.tf
            └── variables.tf
```

## Lambda Function Module

Reusable Lambda module:

```hcl
# modules/lambda-api/main.tf
resource "aws_lambda_function" "this" {
  filename         = var.deployment_package
  function_name    = var.function_name
  role             = aws_iam_role.this.arn
  handler          = var.handler
  source_code_hash = filebase64sha256(var.deployment_package)
  runtime          = var.runtime
  timeout          = var.timeout
  memory_size      = var.memory_size

  environment {
    variables = var.environment_variables
  }

  dynamic "vpc_config" {
    for_each = var.vpc_config != null ? [var.vpc_config] : []

    content {
      subnet_ids         = vpc_config.value.subnet_ids
      security_group_ids = vpc_config.value.security_group_ids
    }
  }

  tracing_config {
    mode = var.xray_tracing_enabled ? "Active" : "PassThrough"
  }

  tags = merge(
    var.tags,
    {
      Name = var.function_name
    }
  )
}

resource "aws_iam_role" "this" {
  name = "${var.function_name}-role"

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

  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "basic_execution" {
  role       = aws_iam_role.this.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy_attachment" "vpc_execution" {
  count = var.vpc_config != null ? 1 : 0

  role       = aws_iam_role.this.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

resource "aws_iam_role_policy" "additional_policies" {
  for_each = var.iam_policy_statements

  name = each.key
  role = aws_iam_role.this.id

  policy = jsonencode({
    Version   = "2012-10-17"
    Statement = [each.value]
  })
}

resource "aws_cloudwatch_log_group" "this" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = var.log_retention_days

  tags = var.tags
}
```

Module variables:

```hcl
# modules/lambda-api/variables.tf
variable "function_name" {
  description = "Name of the Lambda function"
  type        = string
}

variable "deployment_package" {
  description = "Path to the deployment package (.zip file)"
  type        = string
}

variable "handler" {
  description = "Lambda function handler"
  type        = string
  default     = "bootstrap"
}

variable "runtime" {
  description = "Lambda runtime"
  type        = string
  default     = "provided.al2023"
}

variable "timeout" {
  description = "Function timeout in seconds"
  type        = number
  default     = 30
}

variable "memory_size" {
  description = "Function memory in MB"
  type        = number
  default     = 512
}

variable "environment_variables" {
  description = "Environment variables for the function"
  type        = map(string)
  default     = {}
}

variable "vpc_config" {
  description = "VPC configuration for the function"
  type = object({
    subnet_ids         = list(string)
    security_group_ids = list(string)
  })
  default = null
}

variable "xray_tracing_enabled" {
  description = "Enable X-Ray tracing"
  type        = bool
  default     = true
}

variable "log_retention_days" {
  description = "CloudWatch log retention in days"
  type        = number
  default     = 7
}

variable "iam_policy_statements" {
  description = "Additional IAM policy statements"
  type        = map(any)
  default     = {}
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

Module outputs:

```hcl
# modules/lambda-api/outputs.tf
output "function_name" {
  description = "Name of the Lambda function"
  value       = aws_lambda_function.this.function_name
}

output "function_arn" {
  description = "ARN of the Lambda function"
  value       = aws_lambda_function.this.arn
}

output "invoke_arn" {
  description = "Invoke ARN for API Gateway integration"
  value       = aws_lambda_function.this.invoke_arn
}

output "role_arn" {
  description = "ARN of the IAM role"
  value       = aws_iam_role.this.arn
}

output "role_name" {
  description = "Name of the IAM role"
  value       = aws_iam_role.this.name
}

output "log_group_name" {
  description = "Name of the CloudWatch log group"
  value       = aws_cloudwatch_log_group.this.name
}
```

## Using Modules

Call module from root configuration:

```hcl
# env/library-service/dev/main.tf
module "api_lambda" {
  source = "../../../modules/lambda-api"

  function_name      = "${var.service_name}-api-${var.environment}"
  deployment_package = "${var.build_path}/api.zip"
  handler            = "bootstrap"
  runtime            = "provided.al2023"
  timeout            = 30
  memory_size        = 512

  environment_variables = {
    DYNAMODB_TABLE_NAME = aws_dynamodb_table.main.name
    ENVIRONMENT         = var.environment
    LOG_LEVEL           = var.log_level
  }

  iam_policy_statements = {
    dynamodb = {
      Effect = "Allow"
      Action = [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query"
      ]
      Resource = [
        aws_dynamodb_table.main.arn,
        "${aws_dynamodb_table.main.arn}/index/*"
      ]
    }
  }

  tags = {
    Environment = var.environment
    Service     = var.service_name
  }
}

# Use module outputs
output "api_lambda_arn" {
  value = module.api_lambda.function_arn
}
```

## DynamoDB Table Module

Reusable table module:

```hcl
# modules/dynamodb-single-table/main.tf
resource "aws_dynamodb_table" "this" {
  name         = var.table_name
  billing_mode = var.billing_mode
  hash_key     = "PK"
  range_key    = "SK"

  dynamic "attribute" {
    for_each = local.all_attributes

    content {
      name = attribute.value.name
      type = attribute.value.type
    }
  }

  dynamic "global_secondary_index" {
    for_each = var.global_secondary_indexes

    content {
      name            = global_secondary_index.value.name
      hash_key        = global_secondary_index.value.hash_key
      range_key       = global_secondary_index.value.range_key
      projection_type = global_secondary_index.value.projection_type
      read_capacity   = var.billing_mode == "PROVISIONED" ? global_secondary_index.value.read_capacity : null
      write_capacity  = var.billing_mode == "PROVISIONED" ? global_secondary_index.value.write_capacity : null
    }
  }

  ttl {
    attribute_name = var.ttl_attribute_name
    enabled        = var.ttl_enabled
  }

  point_in_time_recovery {
    enabled = var.point_in_time_recovery_enabled
  }

  stream_enabled   = var.stream_enabled
  stream_view_type = var.stream_enabled ? var.stream_view_type : null

  tags = merge(
    var.tags,
    {
      Name = var.table_name
    }
  )
}

locals {
  # Base attributes
  base_attributes = [
    { name = "PK", type = "S" },
    { name = "SK", type = "S" }
  ]

  # GSI attributes
  gsi_attributes = flatten([
    for gsi in var.global_secondary_indexes : [
      { name = gsi.hash_key, type = "S" },
      { name = gsi.range_key, type = "S" }
    ]
  ])

  # Combine and deduplicate
  all_attributes = distinct(concat(local.base_attributes, local.gsi_attributes))
}
```

Module variables:

```hcl
# modules/dynamodb-single-table/variables.tf
variable "table_name" {
  description = "Name of the DynamoDB table"
  type        = string
}

variable "billing_mode" {
  description = "Billing mode (PROVISIONED or PAY_PER_REQUEST)"
  type        = string
  default     = "PAY_PER_REQUEST"
}

variable "global_secondary_indexes" {
  description = "List of global secondary indexes"
  type = list(object({
    name            = string
    hash_key        = string
    range_key       = string
    projection_type = string
    read_capacity   = optional(number)
    write_capacity  = optional(number)
  }))
  default = []
}

variable "ttl_enabled" {
  description = "Enable TTL"
  type        = bool
  default     = true
}

variable "ttl_attribute_name" {
  description = "TTL attribute name"
  type        = string
  default     = "ExpiresAt"
}

variable "point_in_time_recovery_enabled" {
  description = "Enable point-in-time recovery"
  type        = bool
  default     = false
}

variable "stream_enabled" {
  description = "Enable DynamoDB streams"
  type        = bool
  default     = false
}

variable "stream_view_type" {
  description = "Stream view type"
  type        = string
  default     = "NEW_AND_OLD_IMAGES"
}

variable "tags" {
  description = "Tags to apply to the table"
  type        = map(string)
  default     = {}
}
```

Using the table module:

```hcl
module "database" {
  source = "../../../modules/dynamodb-single-table"

  table_name   = "${var.service_name}-${var.environment}"
  billing_mode = "PAY_PER_REQUEST"

  global_secondary_indexes = [
    {
      name            = "GSI1"
      hash_key        = "GSI1PK"
      range_key       = "GSI1SK"
      projection_type = "ALL"
    },
    {
      name            = "GSI2"
      hash_key        = "GSI2PK"
      range_key       = "GSI2SK"
      projection_type = "ALL"
    }
  ]

  point_in_time_recovery_enabled = var.environment == "prod"
  stream_enabled                  = true

  tags = {
    Environment = var.environment
    Service     = var.service_name
  }
}
```

## API Gateway Module

Complete API setup:

```hcl
# modules/api-gateway-lambda/main.tf
resource "aws_apigatewayv2_api" "this" {
  name          = var.api_name
  protocol_type = "HTTP"

  cors_configuration {
    allow_origins = var.cors_allow_origins
    allow_methods = var.cors_allow_methods
    allow_headers = var.cors_allow_headers
    max_age       = var.cors_max_age
  }

  tags = var.tags
}

resource "aws_apigatewayv2_stage" "this" {
  api_id      = aws_apigatewayv2_api.this.id
  name        = var.stage_name
  auto_deploy = true

  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api.arn
    format = jsonencode({
      requestId      = "$context.requestId"
      ip             = "$context.identity.sourceIp"
      requestTime    = "$context.requestTime"
      httpMethod     = "$context.httpMethod"
      routeKey       = "$context.routeKey"
      status         = "$context.status"
      protocol       = "$context.protocol"
      responseLength = "$context.responseLength"
      errorMessage   = "$context.error.message"
    })
  }

  tags = var.tags
}

resource "aws_apigatewayv2_integration" "lambda" {
  api_id             = aws_apigatewayv2_api.this.id
  integration_type   = "AWS_PROXY"
  integration_uri    = var.lambda_invoke_arn
  integration_method = "POST"
}

resource "aws_apigatewayv2_route" "default" {
  api_id    = aws_apigatewayv2_api.this.id
  route_key = "$default"
  target    = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}

resource "aws_lambda_permission" "api_gateway" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = var.lambda_function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.this.execution_arn}/*/*"
}

resource "aws_cloudwatch_log_group" "api" {
  name              = "/aws/apigateway/${var.api_name}"
  retention_in_days = var.log_retention_days

  tags = var.tags
}

# Custom domain (optional)
resource "aws_apigatewayv2_domain_name" "this" {
  count = var.custom_domain_name != null ? 1 : 0

  domain_name = var.custom_domain_name

  domain_name_configuration {
    certificate_arn = var.certificate_arn
    endpoint_type   = "REGIONAL"
    security_policy = "TLS_1_2"
  }

  tags = var.tags
}

resource "aws_apigatewayv2_api_mapping" "this" {
  count = var.custom_domain_name != null ? 1 : 0

  api_id      = aws_apigatewayv2_api.this.id
  domain_name = aws_apigatewayv2_domain_name.this[0].id
  stage       = aws_apigatewayv2_stage.this.id
}
```

Using API Gateway module:

```hcl
module "api" {
  source = "../../../modules/api-gateway-lambda"

  api_name             = "${var.service_name}-${var.environment}"
  stage_name           = var.environment
  lambda_invoke_arn    = module.api_lambda.invoke_arn
  lambda_function_name = module.api_lambda.function_name

  cors_allow_origins = var.environment == "prod" ? ["https://app.example.com"] : ["*"]
  cors_allow_methods = ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
  cors_allow_headers = ["Content-Type", "Authorization"]

  custom_domain_name = var.custom_domain_name
  certificate_arn    = var.certificate_arn

  tags = {
    Environment = var.environment
    Service     = var.service_name
  }
}
```

## Module Versioning

Version modules with Git tags:

```hcl
# Reference specific version
module "lambda" {
  source = "git::https://github.com/org/terraform-modules.git//lambda-api?ref=v1.2.0"

  function_name = "my-function"
  # ...
}

# Reference branch
module "lambda" {
  source = "git::https://github.com/org/terraform-modules.git//lambda-api?ref=main"
}

# Local module (development)
module "lambda" {
  source = "../../../modules/lambda-api"
}
```

## Module Composition

Combine modules:

```hcl
# High-level module combining Lambda + API Gateway
module "serverless_api" {
  source = "../../../modules/serverless-api"

  service_name  = var.service_name
  environment   = var.environment
  build_path    = var.build_path

  # Lambda configuration
  lambda_timeout     = 30
  lambda_memory_size = 512

  # API Gateway configuration
  cors_allow_origins = ["*"]

  # Database
  dynamodb_table_arn = aws_dynamodb_table.main.arn

  tags = local.common_tags
}
```

## Testing Modules

Example configuration for testing:

```hcl
# modules/lambda-api/examples/basic/main.tf
provider "aws" {
  region = "us-east-1"
}

module "example" {
  source = "../.."

  function_name      = "test-lambda"
  deployment_package = "${path.module}/test.zip"

  environment_variables = {
    TEST_VAR = "test_value"
  }

  tags = {
    Test = "true"
  }
}

output "function_arn" {
  value = module.example.function_arn
}
```

## Guidelines

**Module Design:**
- Single responsibility (one logical component)
- Flexible with sensible defaults
- Well-documented inputs and outputs
- Version modules with semantic versioning

**Variables:**
- Required variables have no defaults
- Optional variables have sensible defaults
- Use validation rules where appropriate
- Document all variables

**Outputs:**
- Export all useful attributes
- Use descriptive output names
- Document what each output represents

**File Organization:**
- main.tf for primary resources
- variables.tf for inputs
- outputs.tf for outputs
- versions.tf for provider requirements
- README.md for documentation

**Reusability:**
- Avoid hardcoded values
- Use variables for configuration
- Support multiple use cases
- Keep modules focused and composable

## Benefits

DRY. Write once, use many times.

Consistency. Same pattern across environments.

Testability. Test modules independently.

Maintainability. Update in one place.

## Related

- [terraform-basics.md](./terraform-basics.md) - Fundamentals
- [terraform-best-practices.md](./terraform-best-practices.md) - Advanced patterns
- [aws-lambda-infrastructure.md](../02-aws-lambda/aws-lambda-infrastructure.md) - Lambda patterns
