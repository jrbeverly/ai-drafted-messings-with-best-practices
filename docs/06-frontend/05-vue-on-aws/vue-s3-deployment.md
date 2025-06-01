# Vue SPA Deployment to S3

Deploy Vue single-page applications to Amazon S3 with optimized caching, automated CI/CD, and multi-environment support.

**Keywords:** vue-spa, s3-deployment, vite-build, static-hosting, cache-control, ci-cd, aws-s3-sync, multi-environment, iam-policy, fingerprinted-assets

## Principle

S3 static website hosting is the ideal deployment target for Vue SPAs built with Vite. The build output consists entirely of static files that S3 serves directly without a running server. Pair precise cache-control headers with content-hashed filenames to achieve both instant cache invalidation for new releases and aggressive long-term caching for unchanged assets. This yields zero-cost-when-idle hosting with near-instant global delivery when fronted by CloudFront.

## Vite Build Output Structure

Vite produces a deterministic output structure where all assets except `index.html` receive content-based hashes in their filenames:

```
dist/
├── index.html                          # Entry point, references hashed assets
├── assets/
│   ├── index-3a8b2c1d.js             # Main JS bundle (content-hashed)
│   ├── index-7f4e9a2b.css            # Main CSS bundle (content-hashed)
│   ├── vendor-9c1d4e3f.js            # Vendor chunk (content-hashed)
│   ├── AboutView-2b5c8d1a.js         # Lazy-loaded route chunk
│   └── logo-a1b2c3d4.svg            # Static assets (content-hashed)
└── favicon.ico                         # Root-level static files
```

Key insight: `index.html` is the only file that changes its content without changing its name. Every other file has a unique hash. This drives the entire caching strategy.

Configure the Vite build for production:

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  build: {
    // Output directory (default is 'dist')
    outDir: 'dist',
    // Generate source maps for error tracking (do not serve publicly)
    sourcemap: 'hidden',
    // Chunk splitting strategy
    rollupOptions: {
      output: {
        // Separate vendor chunks for better caching
        manualChunks: {
          'vue-vendor': ['vue', 'vue-router', 'pinia'],
          'ui-vendor': ['vuetify']
        }
      }
    }
  }
})
```

## S3 Bucket Configuration

Create an S3 bucket configured for static website hosting:

```hcl
# env/my-app/main.tf

resource "aws_s3_bucket" "spa" {
  bucket = "${var.project_name}-${var.environment}-spa"

  tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
  }
}

# Block all public access when using CloudFront OAC (recommended)
resource "aws_s3_bucket_public_access_block" "spa" {
  bucket = aws_s3_bucket.spa.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Bucket policy granting CloudFront access via OAC
resource "aws_s3_bucket_policy" "spa" {
  bucket = aws_s3_bucket.spa.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AllowCloudFrontOAC"
        Effect    = "Allow"
        Principal = {
          Service = "cloudfront.amazonaws.com"
        }
        Action   = "s3:GetObject"
        Resource = "${aws_s3_bucket.spa.arn}/*"
        Condition = {
          StringEquals = {
            "AWS:SourceArn" = aws_cloudfront_distribution.spa.arn
          }
        }
      }
    ]
  })
}

# Enable versioning for rollback capability
resource "aws_s3_bucket_versioning" "spa" {
  bucket = aws_s3_bucket.spa.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Lifecycle rule to clean up old versions after 30 days
resource "aws_s3_bucket_lifecycle_configuration" "spa" {
  bucket = aws_s3_bucket.spa.id

  rule {
    id     = "cleanup-old-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 30
    }
  }
}
```

For direct S3 website hosting without CloudFront (development environments only):

```hcl
# Only for non-production environments
resource "aws_s3_bucket_website_configuration" "spa_dev" {
  count  = var.environment == "dev" ? 1 : 0
  bucket = aws_s3_bucket.spa.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "index.html"
  }
}
```

## IAM Deployment Policy

Create a least-privilege IAM policy for the deployment pipeline:

```hcl
# env/my-app/iam.tf

resource "aws_iam_policy" "spa_deploy" {
  name        = "${var.project_name}-${var.environment}-spa-deploy"
  description = "Allows CI/CD to deploy SPA files to S3 and invalidate CloudFront"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "S3Upload"
        Effect = "Allow"
        Action = [
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.spa.arn,
          "${aws_s3_bucket.spa.arn}/*"
        ]
      },
      {
        Sid    = "CloudFrontInvalidation"
        Effect = "Allow"
        Action = [
          "cloudfront:CreateInvalidation",
          "cloudfront:GetInvalidation"
        ]
        Resource = aws_cloudfront_distribution.spa.arn
      }
    ]
  })
}

# OIDC role for GitHub Actions (no long-lived credentials)
resource "aws_iam_role" "github_actions" {
  name = "${var.project_name}-${var.environment}-github-actions"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/token.actions.githubusercontent.com"
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          }
          StringLike = {
            "token.actions.githubusercontent.com:sub" = "repo:${var.github_org}/${var.github_repo}:ref:refs/heads/${var.deploy_branch}"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "github_actions_deploy" {
  role       = aws_iam_role.github_actions.name
  policy_arn = aws_iam_policy.spa_deploy.arn
}
```

## Deployment Script

The deployment script uploads files with appropriate cache-control headers based on file type:

```bash
#!/usr/bin/env bash
# scripts/deploy-spa.sh
# Deploy Vue SPA to S3 with optimized cache-control headers
set -euo pipefail

BUCKET_NAME="${1:?Usage: deploy-spa.sh <bucket-name> [distribution-id]}"
DISTRIBUTION_ID="${2:-}"
BUILD_DIR="dist"

if [ ! -d "$BUILD_DIR" ]; then
  echo "Error: Build directory '$BUILD_DIR' not found. Run 'npm run build' first."
  exit 1
fi

echo "Deploying to s3://${BUCKET_NAME}..."

# Step 1: Upload fingerprinted assets with immutable cache (1 year)
# These files have content hashes in their names and never change
aws s3 sync "$BUILD_DIR/assets/" "s3://${BUCKET_NAME}/assets/" \
  --cache-control "public, max-age=31536000, immutable" \
  --delete

# Step 2: Upload index.html with short cache (no-cache forces revalidation)
# This ensures users always get the latest version that references new assets
aws s3 cp "$BUILD_DIR/index.html" "s3://${BUCKET_NAME}/index.html" \
  --cache-control "no-cache, no-store, must-revalidate" \
  --content-type "text/html"

# Step 3: Upload remaining root-level files (favicon, robots.txt, etc.)
# Short cache since these files are not fingerprinted
aws s3 sync "$BUILD_DIR/" "s3://${BUCKET_NAME}/" \
  --exclude "assets/*" \
  --exclude "index.html" \
  --cache-control "public, max-age=3600" \
  --delete

# Step 4: Invalidate CloudFront cache for index.html (if distribution provided)
if [ -n "$DISTRIBUTION_ID" ]; then
  echo "Invalidating CloudFront distribution ${DISTRIBUTION_ID}..."
  INVALIDATION_ID=$(aws cloudfront create-invalidation \
    --distribution-id "$DISTRIBUTION_ID" \
    --paths "/index.html" "/" \
    --query 'Invalidation.Id' \
    --output text)
  echo "Invalidation created: ${INVALIDATION_ID}"

  # Wait for invalidation to complete
  echo "Waiting for invalidation to complete..."
  aws cloudfront wait invalidation-completed \
    --distribution-id "$DISTRIBUTION_ID" \
    --id "$INVALIDATION_ID"
  echo "Invalidation complete."
fi

echo "Deployment successful."
```

Make the script executable:

```bash
chmod +x scripts/deploy-spa.sh
```

## Cache-Control Strategy

The caching strategy exploits Vite's content-hashing to maximize cache hits while guaranteeing users receive new code on every deployment:

| File Type | Cache-Control Header | TTL | Rationale |
|---|---|---|---|
| `index.html` | `no-cache, no-store, must-revalidate` | 0 (always revalidate) | Entry point references hashed assets; must always be fresh |
| `assets/*.js` | `public, max-age=31536000, immutable` | 1 year | Content hash in filename; file never changes |
| `assets/*.css` | `public, max-age=31536000, immutable` | 1 year | Content hash in filename; file never changes |
| `assets/*.svg/png/woff2` | `public, max-age=31536000, immutable` | 1 year | Content hash in filename; file never changes |
| `favicon.ico` | `public, max-age=3600` | 1 hour | No content hash; short cache allows periodic updates |
| `robots.txt` | `public, max-age=3600` | 1 hour | No content hash; infrequent changes |

How the flow works:

1. Browser requests `/` and receives `index.html` (always fresh due to `no-cache`)
2. `index.html` contains `<script src="/assets/index-3a8b2c1d.js">`
3. Browser requests the hashed JS file, which is cached for 1 year
4. On next deploy, Vite produces `index-NEW_HASH.js`
5. Fresh `index.html` references the new hash; old cached JS is never requested again

## CI/CD Pipeline

GitHub Actions workflow for automated deployment:

```yaml
# .github/workflows/deploy-spa.yml
name: Deploy SPA

on:
  push:
    branches: [main]
    paths:
      - 'app/MyApp/**'
      - '.github/workflows/deploy-spa.yml'

permissions:
  id-token: write   # Required for OIDC
  contents: read

env:
  NODE_VERSION: '20'
  WORKING_DIR: 'app/MyApp'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: '${{ env.WORKING_DIR }}/package-lock.json'

      - name: Install dependencies
        working-directory: ${{ env.WORKING_DIR }}
        run: npm ci

      - name: Run type check
        working-directory: ${{ env.WORKING_DIR }}
        run: npm run type-check

      - name: Run tests
        working-directory: ${{ env.WORKING_DIR }}
        run: npm run test:unit -- --run

      - name: Build
        working-directory: ${{ env.WORKING_DIR }}
        run: npm run build
        env:
          VITE_API_BASE_URL: ${{ vars.API_BASE_URL }}
          VITE_APP_ENV: production

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Deploy to S3
        working-directory: ${{ env.WORKING_DIR }}
        run: |
          bash ../../scripts/deploy-spa.sh \
            "${{ vars.S3_BUCKET_NAME }}" \
            "${{ vars.CLOUDFRONT_DISTRIBUTION_ID }}"
```

## Multi-Environment S3 Buckets

Support multiple environments with separate buckets and configurations:

```hcl
# env/my-app/variables.tf

variable "environments" {
  description = "Map of environment configurations"
  type = map(object({
    domain_name  = string
    deploy_branch = string
  }))
  default = {
    dev = {
      domain_name  = "dev.myapp.example.com"
      deploy_branch = "develop"
    }
    staging = {
      domain_name  = "staging.myapp.example.com"
      deploy_branch = "staging"
    }
    production = {
      domain_name  = "myapp.example.com"
      deploy_branch = "main"
    }
  }
}
```

```hcl
# env/my-app/main.tf

resource "aws_s3_bucket" "spa" {
  for_each = var.environments

  bucket = "${var.project_name}-${each.key}-spa"

  tags = {
    Environment = each.key
    Project     = var.project_name
    ManagedBy   = "terraform"
  }
}
```

Manage environment-specific Vite configuration through `.env` files:

```bash
# .env.development
VITE_API_BASE_URL=https://dev-api.myapp.example.com
VITE_APP_ENV=development

# .env.staging
VITE_API_BASE_URL=https://staging-api.myapp.example.com
VITE_APP_ENV=staging

# .env.production
VITE_API_BASE_URL=https://api.myapp.example.com
VITE_APP_ENV=production
```

Build for a specific environment:

```bash
# Vite loads .env.staging automatically with --mode
npx vite build --mode staging
```

## Best Practices

**DO:**
- Use content-hashed filenames for all assets (Vite default)
- Set `immutable` cache-control on fingerprinted assets
- Always revalidate `index.html` with `no-cache`
- Use OIDC federation for CI/CD AWS credentials (no long-lived keys)
- Enable S3 versioning for rollback capability
- Separate deployment into distinct upload steps per cache policy
- Invalidate only `index.html` and `/` in CloudFront after deployment
- Use `--delete` flag in `aws s3 sync` to remove orphaned files

**DON'T:**
- Set long cache headers on `index.html` (users will see stale content)
- Use a single `aws s3 sync` with one cache-control for all files
- Store AWS credentials as GitHub Actions secrets (use OIDC roles)
- Skip the build step type check and test run in CI/CD
- Upload source maps to the public S3 bucket
- Enable S3 static website hosting in production (use CloudFront OAC instead)
- Invalidate `/*` in CloudFront (expensive; invalidate only what changed)

## Guidelines

**Essential:**
- Configure Vite build with content hashing enabled (default behavior)
- Implement per-file-type cache-control headers in the deployment script
- Block public access on S3 and serve exclusively through CloudFront OAC
- Use OIDC for CI/CD authentication with AWS

**Recommended:**
- Split vendor dependencies into a separate chunk for stable caching
- Enable S3 versioning and set lifecycle rules to expire old versions
- Run type checking, linting, and tests before building in CI/CD
- Use GitHub Actions environments with required approvals for production

**Advanced:**
- Implement blue/green deployments by swapping CloudFront origin paths
- Add deployment notifications to Slack or other channels
- Monitor deployment success with CloudWatch metrics on 4xx/5xx rates
- Generate and upload source maps to an error tracking service separately

## Benefits

Zero-cost-when-idle. S3 charges only for storage and requests; no running servers.

Instant cache invalidation. Content-hashed assets guarantee users get new code without waiting for cache expiry.

Global availability. CloudFront edge caching delivers assets from the nearest location.

Immutable deployments. S3 versioning enables instant rollback to any previous deployment.

Minimal attack surface. No server to patch, no runtime vulnerabilities, no SSH access.

Automated and reproducible. CI/CD pipeline ensures consistent, tested deployments every time.

## Related

- [spa-routing-s3-cloudfront.md](./spa-routing-s3-cloudfront.md) - Handle SPA client-side routing on S3 and CloudFront
- [vue-cloudfront-optimization.md](./vue-cloudfront-optimization.md) - CloudFront cache strategies and security headers for Vue SPAs
