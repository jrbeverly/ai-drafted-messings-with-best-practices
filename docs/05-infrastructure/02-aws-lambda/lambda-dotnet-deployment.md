# Lambda .NET Deployment

Deploy .NET applications to AWS Lambda. Native AOT, container images, and layer patterns.

## Principle

Use native AOT for fastest cold starts. Container images for large dependencies. Layers for shared code.

## Project Setup

Lambda project structure:

```
src/LibraryService/
├── LibraryService.Lambda/
│   ├── Function.cs
│   ├── Startup.cs
│   ├── LibraryService.Lambda.csproj
│   └── aws-lambda-tools-defaults.json
├── LibraryService.Application/
│   └── ...
├── LibraryService.Domain/
│   └── ...
└── LibraryService.Infrastructure/
    └── ...
```

Project file:

```xml
<!-- LibraryService.Lambda.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <GenerateRuntimeConfigurationFiles>true</GenerateRuntimeConfigurationFiles>
    <AWSProjectType>Lambda</AWSProjectType>

    <!-- Native AOT for faster cold starts -->
    <PublishAot>true</PublishAot>
    <InvariantGlobalization>true</InvariantGlobalization>
    <StripSymbols>true</StripSymbols>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Amazon.Lambda.Core" Version="2.2.0" />
    <PackageReference Include="Amazon.Lambda.Serialization.SystemTextJson" Version="2.4.0" />
    <PackageReference Include="Amazon.Lambda.APIGatewayEvents" Version="2.7.0" />
    <PackageReference Include="AWSSDK.DynamoDBv2" Version="3.7.300" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="8.0.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\LibraryService.Application\LibraryService.Application.csproj" />
    <ProjectReference Include="..\LibraryService.Infrastructure\LibraryService.Infrastructure.csproj" />
  </ItemGroup>
</Project>
```

Lambda tools configuration:

```json
// aws-lambda-tools-defaults.json
{
  "Information": [
    "This file provides default values for the deployment wizard inside Visual Studio and the AWS Lambda commands added to the .NET Core CLI.",
    "To learn more about the Lambda commands with the .NET Core CLI execute the following command at the command line in the project root directory.",
    "dotnet lambda help",
    "All the command line options for the Lambda command can be specified in this file."
  ],
  "profile": "default",
  "region": "us-east-1",
  "configuration": "Release",
  "function-runtime": "dotnet8",
  "function-memory-size": 512,
  "function-timeout": 30,
  "function-handler": "LibraryService.Lambda::LibraryService.Lambda.Function::FunctionHandler",
  "framework": "net8.0",
  "function-architecture": "arm64"
}
```

## Native AOT Deployment

Build and deploy with Native AOT:

```bash
# Install Lambda tools
dotnet tool install -g Amazon.Lambda.Tools

# Build for Native AOT
cd src/LibraryService/LibraryService.Lambda
dotnet publish -c Release

# Package for Lambda
cd bin/Release/net8.0/linux-arm64/publish
zip -r ../../../../../../../deploy/api.zip .

# Deploy with Terraform
cd ../../../../../../env/library-service/dev
terraform apply
```

Automated build script:

```bash
#!/bin/bash
# scripts/build-lambda.sh

set -e

SERVICE=$1
ENVIRONMENT=$2

if [ -z "$SERVICE" ] || [ -z "$ENVIRONMENT" ]; then
  echo "Usage: ./build-lambda.sh <service> <environment>"
  exit 1
fi

echo "Building $SERVICE for $ENVIRONMENT..."

# Clean previous builds
rm -rf build/$SERVICE

# Publish with Native AOT
dotnet publish src/$SERVICE/$SERVICE.Lambda \
  -c Release \
  --runtime linux-arm64 \
  --self-contained \
  -o build/$SERVICE/publish

# Create deployment package
cd build/$SERVICE/publish
zip -r ../../$SERVICE.zip .
cd ../../..

echo "Build complete: build/$SERVICE.zip"
```

## Container Image Deployment

Use container images for large dependencies:

```dockerfile
# Dockerfile
FROM public.ecr.aws/lambda/dotnet:8 AS base

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy project files
COPY ["src/LibraryService/LibraryService.Lambda/LibraryService.Lambda.csproj", "LibraryService.Lambda/"]
COPY ["src/LibraryService/LibraryService.Application/LibraryService.Application.csproj", "LibraryService.Application/"]
COPY ["src/LibraryService/LibraryService.Infrastructure/LibraryService.Infrastructure.csproj", "LibraryService.Infrastructure/"]
COPY ["src/LibraryService/LibraryService.Domain/LibraryService.Domain.csproj", "LibraryService.Domain/"]

# Restore dependencies
RUN dotnet restore "LibraryService.Lambda/LibraryService.Lambda.csproj"

# Copy source code
COPY src/LibraryService/ .

# Build and publish
WORKDIR "/src/LibraryService.Lambda"
RUN dotnet build "LibraryService.Lambda.csproj" -c Release -o /app/build
RUN dotnet publish "LibraryService.Lambda.csproj" -c Release -o /app/publish

# Runtime image
FROM base AS final
WORKDIR /var/task

COPY --from=build /app/publish .

CMD ["LibraryService.Lambda::LibraryService.Lambda.Function::FunctionHandler"]
```

Build and push:

```bash
#!/bin/bash
# scripts/deploy-lambda-container.sh

set -e

AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION="us-east-1"
SERVICE="library-service"
ENVIRONMENT="dev"
IMAGE_TAG="${SERVICE}-${ENVIRONMENT}"
ECR_REPO="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${IMAGE_TAG}"

# Login to ECR
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin $ECR_REPO

# Build image
docker build -t $IMAGE_TAG \
  -f src/LibraryService/LibraryService.Lambda/Dockerfile .

# Tag image
docker tag $IMAGE_TAG:latest $ECR_REPO:latest

# Push to ECR
docker push $ECR_REPO:latest

echo "Image pushed: $ECR_REPO:latest"
```

Terraform configuration:

```hcl
resource "aws_ecr_repository" "lambda" {
  name                 = "${var.service_name}-${var.environment}"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }
}

resource "aws_lambda_function" "api" {
  function_name = "${var.service_name}-api-${var.environment}"
  role          = aws_iam_role.lambda.arn
  timeout       = 30
  memory_size   = 512

  # Use container image
  package_type = "Image"
  image_uri    = "${aws_ecr_repository.lambda.repository_url}:latest"

  environment {
    variables = {
      DYNAMODB_TABLE_NAME = aws_dynamodb_table.main.name
      ENVIRONMENT         = var.environment
    }
  }
}
```

## Lambda Layers

Share dependencies across functions:

```bash
# Create layer directory structure
mkdir -p layers/aws-sdk/lib

# Copy dependencies
dotnet publish src/Common/Common.csproj -c Release -o layers/aws-sdk/lib

# Package layer
cd layers/aws-sdk
zip -r ../aws-sdk-layer.zip .
cd ../..
```

Deploy layer:

```hcl
resource "aws_lambda_layer_version" "aws_sdk" {
  filename            = "${var.build_path}/layers/aws-sdk-layer.zip"
  layer_name          = "${var.service_name}-aws-sdk-${var.environment}"
  source_code_hash    = filebase64sha256("${var.build_path}/layers/aws-sdk-layer.zip")
  compatible_runtimes = ["dotnet8"]
}

resource "aws_lambda_function" "api" {
  function_name = "${var.service_name}-api-${var.environment}"

  layers = [
    aws_lambda_layer_version.aws_sdk.arn
  ]
}
```

## CI/CD Pipeline

GitHub Actions deployment:

```yaml
# .github/workflows/deploy-lambda.yml
name: Deploy Lambda

on:
  push:
    branches:
      - main
    paths:
      - 'src/LibraryService/**'

env:
  DOTNET_VERSION: '8.0'
  AWS_REGION: 'us-east-1'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Install Lambda Tools
        run: dotnet tool install -g Amazon.Lambda.Tools

      - name: Restore dependencies
        run: dotnet restore src/LibraryService/LibraryService.Lambda

      - name: Build
        run: dotnet build src/LibraryService/LibraryService.Lambda -c Release

      - name: Test
        run: dotnet test src/LibraryService/LibraryService.Tests

      - name: Publish
        run: |
          dotnet publish src/LibraryService/LibraryService.Lambda \
            -c Release \
            --runtime linux-arm64 \
            --self-contained \
            -o build/publish

      - name: Package
        run: |
          cd build/publish
          zip -r ../api.zip .
          cd ../..

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0

      - name: Terraform Init
        run: terraform init
        working-directory: env/library-service/dev

      - name: Terraform Apply
        run: terraform apply -auto-approve
        working-directory: env/library-service/dev
```

## Version Management

Lambda versioning:

```hcl
resource "aws_lambda_function" "api" {
  function_name = "${var.service_name}-api-${var.environment}"

  # Publish new version on every deploy
  publish = true
}

# Alias for stable endpoint
resource "aws_lambda_alias" "live" {
  name             = "live"
  function_name    = aws_lambda_function.api.function_name
  function_version = aws_lambda_function.api.version
}

# API Gateway points to alias
resource "aws_apigatewayv2_integration" "lambda" {
  api_id             = aws_apigatewayv2_api.this.id
  integration_type   = "AWS_PROXY"
  integration_uri    = aws_lambda_alias.live.invoke_arn
  integration_method = "POST"
}
```

Blue-green deployment:

```hcl
resource "aws_lambda_alias" "live" {
  name             = "live"
  function_name    = aws_lambda_function.api.function_name
  function_version = aws_lambda_function.api.version

  # Canary deployment: 10% to new version
  routing_config {
    additional_version_weights = {
      (aws_lambda_function.api.version - 1) = 0.9
    }
  }
}
```

## Local Testing

Test Lambda locally:

```bash
# Install AWS SAM CLI
brew install aws-sam-cli

# Create SAM template
cat > template.yaml <<EOF
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  ApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: LibraryService.Lambda::LibraryService.Lambda.Function::FunctionHandler
      Runtime: dotnet8
      CodeUri: ./build/api.zip
      MemorySize: 512
      Timeout: 30
      Environment:
        Variables:
          DYNAMODB_TABLE_NAME: library-service-local
          ENVIRONMENT: local
EOF

# Start local API
sam local start-api

# Invoke function
curl http://localhost:3000/users/123
```

## Performance Optimization

Trim unused dependencies:

```xml
<PropertyGroup>
  <PublishTrimmed>true</PublishTrimmed>
  <TrimMode>link</TrimMode>

  <!-- Trim unused code -->
  <InvariantGlobalization>true</InvariantGlobalization>
  <EventSourceSupport>false</EventSourceSupport>
  <HttpActivityPropagationSupport>false</HttpActivityPropagationSupport>
</PropertyGroup>
```

ReadyToRun for faster startup:

```xml
<PropertyGroup>
  <PublishReadyToRun>true</PublishReadyToRun>
  <PublishReadyToRunShowWarnings>true</PublishReadyToRunShowWarnings>
</PropertyGroup>
```

## Monitoring Deployment

Check deployment status:

```bash
# Get function info
aws lambda get-function --function-name library-service-api-dev

# Get latest version
aws lambda list-versions-by-function --function-name library-service-api-dev

# Get alias info
aws lambda get-alias --function-name library-service-api-dev --name live

# Invoke function
aws lambda invoke \
  --function-name library-service-api-dev \
  --payload '{"httpMethod":"GET","path":"/users/123"}' \
  response.json

# View logs
aws logs tail /aws/lambda/library-service-api-dev --follow
```

## Guidelines

**Deployment Package:**
- Use Native AOT for fastest cold starts
- Container images for large dependencies (>50 MB)
- Layers for shared dependencies across functions
- Keep package size under 250 MB (50 MB zipped)

**Build Process:**
- Automate builds in CI/CD
- Run tests before deployment
- Use semantic versioning
- Tag releases in Git

**Environment Management:**
- Separate dev/staging/prod
- Use environment variables for configuration
- Never hardcode secrets
- Use parameter store or secrets manager

**Versioning:**
- Publish versions for rollback capability
- Use aliases for stable endpoints
- Implement canary deployments for production
- Keep last 3-5 versions

**Testing:**
- Unit tests for business logic
- Integration tests with LocalStack or SAM
- Load testing before production
- Monitor cold start frequency

## Benefits

Fast cold starts. Native AOT optimizes startup.

Small packages. Trimmed dependencies reduce size.

Automated. CI/CD handles builds and deployments.

Safe. Versioning enables instant rollback.

## Related

- [lambda-best-practices.md](./lambda-best-practices.md) - Optimization patterns
- [terraform-modules.md](../01-terraform/terraform-modules.md) - Infrastructure modules
- [minimal-api.md](../../03-csharp/01-core/minimal-api.md) - API patterns
