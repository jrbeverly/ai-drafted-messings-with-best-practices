# Hugo CI/CD on AWS

Gitea Actions Hugo pipeline. Automated build and deploy. S3 sync. CloudFront invalidation. Multi-environment deployment.

## Principle

Automate Hugo site builds and deployments with CI/CD pipelines. Build on push, deploy to S3, invalidate CloudFront cache. Use Gitea Actions (or GitHub Actions compatible runners) for a self-hosted, zero-cost CI/CD workflow.

## Pipeline Overview

```
Push to main → Build Hugo → Run Tests → Deploy to S3 → Invalidate CloudFront
     │
     └─ Push to develop → Build → Deploy to Staging
```

### Deployment Flow

1. Developer pushes to Git
2. CI pipeline triggers
3. Hugo builds site with `--minify`
4. HTML validation / link checking (optional)
5. Sync to S3 with proper cache headers
6. CloudFront cache invalidation
7. Smoke test (verify site is live)

## Gitea Actions Workflow

### Basic Build and Deploy

**.gitea/workflows/deploy.yml:**

```yaml
name: Deploy Hugo Site

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0  # Needed for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build
        run: hugo --minify --gc --cleanDestinationDir

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Deploy to S3
        run: |
          aws s3 sync public/ s3://${{ secrets.S3_BUCKET }}/ \
            --delete \
            --cache-control "public, max-age=300, must-revalidate" \
            --exclude "*.css" \
            --exclude "*.js" \
            --exclude "*.woff*" \
            --exclude "*.ttf" \
            --exclude "*.eot" \
            --exclude "*.jpg" \
            --exclude "*.jpeg" \
            --exclude "*.png" \
            --exclude "*.gif" \
            --exclude "*.svg" \
            --exclude "*.webp" \
            --exclude "*.avif" \
            --exclude "*.ico"

          # Sync CSS/JS with immutable cache
          aws s3 sync public/ s3://${{ secrets.S3_BUCKET }}/ \
            --exclude "*" \
            --include "*.css" \
            --include "*.js" \
            --cache-control "public, max-age=31536000, immutable"

          # Sync images with medium cache
          aws s3 sync public/ s3://${{ secrets.S3_BUCKET }}/ \
            --exclude "*" \
            --include "*.jpg" \
            --include "*.jpeg" \
            --include "*.png" \
            --include "*.gif" \
            --include "*.svg" \
            --include "*.webp" \
            --include "*.avif" \
            --include "*.ico" \
            --cache-control "public, max-age=86400"

          # Sync fonts with immutable cache
          aws s3 sync public/ s3://${{ secrets.S3_BUCKET }}/ \
            --exclude "*" \
            --include "*.woff" \
            --include "*.woff2" \
            --include "*.ttf" \
            --include "*.eot" \
            --cache-control "public, max-age=31536000, immutable"

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

### Multi-Environment Pipeline

**.gitea/workflows/deploy-multi-env.yml:**

```yaml
name: Deploy Hugo Site (Multi-Environment)

on:
  push:
    branches:
      - main
      - develop

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true

      - name: Determine Environment
        id: env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "environment=production" >> $GITHUB_OUTPUT
            echo "base_url=https://example.com/" >> $GITHUB_OUTPUT
            echo "s3_bucket=${{ secrets.PROD_S3_BUCKET }}" >> $GITHUB_OUTPUT
            echo "cf_distribution=${{ secrets.PROD_CF_DISTRIBUTION_ID }}" >> $GITHUB_OUTPUT
          else
            echo "environment=staging" >> $GITHUB_OUTPUT
            echo "base_url=https://staging.example.com/" >> $GITHUB_OUTPUT
            echo "s3_bucket=${{ secrets.STAGING_S3_BUCKET }}" >> $GITHUB_OUTPUT
            echo "cf_distribution=${{ secrets.STAGING_CF_DISTRIBUTION_ID }}" >> $GITHUB_OUTPUT
          fi

      - name: Build Hugo
        run: |
          hugo --minify --gc --cleanDestinationDir \
            --baseURL "${{ steps.env.outputs.base_url }}" \
            --environment "${{ steps.env.outputs.environment }}"

      - name: Validate HTML
        run: |
          # Install htmltest
          curl -sL https://github.com/wjdp/htmltest/releases/latest/download/htmltest_linux_amd64.tar.gz | tar xz
          ./htmltest --conf .htmltest.yml || true  # Non-blocking

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Deploy to S3
        run: |
          ./scripts/deploy-s3.sh "${{ steps.env.outputs.s3_bucket }}"

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id "${{ steps.env.outputs.cf_distribution }}" \
            --paths "/*"

      - name: Smoke Test
        run: |
          sleep 30  # Wait for propagation
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "${{ steps.env.outputs.base_url }}")
          if [ "$STATUS" != "200" ]; then
            echo "Smoke test failed: HTTP $STATUS"
            exit 1
          fi
          echo "Smoke test passed: HTTP $STATUS"
```

### Deployment Script

**scripts/deploy-s3.sh:**

```bash
#!/bin/bash
set -euo pipefail

BUCKET="${1:?Usage: deploy-s3.sh <bucket-name>}"
SOURCE="public"

echo "Deploying to s3://${BUCKET}/"

# HTML files - short cache, must revalidate
aws s3 sync "${SOURCE}/" "s3://${BUCKET}/" \
  --exclude "*" \
  --include "*.html" \
  --cache-control "public, max-age=300, must-revalidate" \
  --content-type "text/html; charset=utf-8" \
  --delete

# CSS/JS - immutable (fingerprinted by Hugo)
aws s3 sync "${SOURCE}/" "s3://${BUCKET}/" \
  --exclude "*" \
  --include "*.css" \
  --include "*.js" \
  --cache-control "public, max-age=31536000, immutable"

# Images - 1 day cache
aws s3 sync "${SOURCE}/" "s3://${BUCKET}/" \
  --exclude "*" \
  --include "*.jpg" --include "*.jpeg" \
  --include "*.png" --include "*.gif" \
  --include "*.svg" --include "*.webp" \
  --include "*.avif" --include "*.ico" \
  --cache-control "public, max-age=86400"

# Fonts - immutable
aws s3 sync "${SOURCE}/" "s3://${BUCKET}/" \
  --exclude "*" \
  --include "*.woff" --include "*.woff2" \
  --include "*.ttf" --include "*.eot" \
  --cache-control "public, max-age=31536000, immutable"

# Everything else (XML, JSON, txt, etc.)
aws s3 sync "${SOURCE}/" "s3://${BUCKET}/" \
  --exclude "*.html" \
  --exclude "*.css" --exclude "*.js" \
  --exclude "*.jpg" --exclude "*.jpeg" \
  --exclude "*.png" --exclude "*.gif" \
  --exclude "*.svg" --exclude "*.webp" \
  --exclude "*.avif" --exclude "*.ico" \
  --exclude "*.woff" --exclude "*.woff2" \
  --exclude "*.ttf" --exclude "*.eot" \
  --cache-control "public, max-age=3600" \
  --delete

echo "Deployment complete!"
```

## Hugo Deploy Method

### Using Hugo's Built-in Deploy

**.gitea/workflows/deploy-hugo-native.yml:**

```yaml
name: Deploy Hugo (Native Deploy)

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Build and Deploy
        run: |
          hugo --minify --gc --cleanDestinationDir
          hugo deploy --target production --maxDeletes 100

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

**Hugo config.toml deployment targets:**

```toml
[deployment]
  [[deployment.targets]]
    name = "production"
    URL = "s3://example-com-hugo?region=us-east-1"

  [[deployment.targets]]
    name = "staging"
    URL = "s3://staging-example-com-hugo?region=us-east-1"

  [[deployment.matchers]]
    pattern = "^.+\\.html$"
    cacheControl = "public, max-age=300, must-revalidate"
    contentType = "text/html; charset=utf-8"
    gzip = true

  [[deployment.matchers]]
    pattern = "^.+\\.(css|js)$"
    cacheControl = "public, max-age=31536000, immutable"
    gzip = true

  [[deployment.matchers]]
    pattern = "^.+\\.(png|jpg|jpeg|gif|svg|webp|avif)$"
    cacheControl = "public, max-age=86400"

  [[deployment.matchers]]
    pattern = "^.+\\.(woff|woff2|ttf|eot)$"
    cacheControl = "public, max-age=31536000, immutable"

  [[deployment.matchers]]
    pattern = "^.+\\.(xml|json|txt)$"
    cacheControl = "public, max-age=3600"
    gzip = true
```

## OIDC Authentication (Recommended)

### Use OIDC Instead of Access Keys

**Why OIDC?** No long-lived credentials stored as secrets. Temporary tokens generated per workflow run.

**IAM Role Trust Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/gitea.example.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "gitea.example.com:aud": "https://gitea.example.com",
          "gitea.example.com:sub": "repo:myorg/hugo-site:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**Workflow with OIDC:**

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/hugo-deploy-role
          aws-region: us-east-1

      # ... build and deploy steps
```

## Build Validation

### HTML Validation

**.htmltest.yml:**

```yaml
DirectoryPath: "public"
CheckExternal: false
CheckInternal: true
CheckMailto: false
CheckFavicon: true
EnforceHTTPS: true
IgnoreDirectoryMissingTrailingSlash: true
IgnoreURLs:
  - "example.com"
  - "localhost"
```

### Link Checking

```yaml
      - name: Check Links
        run: |
          # Install muffet (fast link checker)
          go install github.com/raviqqe/muffet/v2@latest

          # Start local server
          hugo server --minify --port 1313 &
          sleep 5

          # Check internal links
          muffet http://localhost:1313/ \
            --exclude="example.com" \
            --timeout=30 \
            --rate-limit=10 || true

          kill %1
```

### Lighthouse CI

```yaml
      - name: Lighthouse CI
        run: |
          npm install -g @lhci/cli
          hugo server --minify --port 1313 &
          sleep 5

          lhci autorun --config=.lighthouserc.json || true
          kill %1
```

**.lighthouserc.json:**

```json
{
  "ci": {
    "collect": {
      "url": ["http://localhost:1313/", "http://localhost:1313/blog/"],
      "numberOfRuns": 3
    },
    "assert": {
      "assertions": {
        "categories:performance": ["warn", {"minScore": 0.9}],
        "categories:accessibility": ["error", {"minScore": 0.9}],
        "categories:best-practices": ["warn", {"minScore": 0.9}],
        "categories:seo": ["warn", {"minScore": 0.9}]
      }
    }
  }
}
```

## Hugo Environments

### Environment-Specific Config

**config/production/config.toml:**

```toml
baseURL = "https://example.com/"
enableRobotsTXT = true

[params]
  env = "production"
  analytics_id = "G-XXXXXXXXXX"
```

**config/staging/config.toml:**

```toml
baseURL = "https://staging.example.com/"
enableRobotsTXT = false

[params]
  env = "staging"
  noindex = true
```

**Build with environment:**

```bash
# Production
hugo --minify --environment production

# Staging
hugo --minify --environment staging
```

### Staging noindex

**layouts/partials/head/meta.html:**

```go-html-template
{{ if .Site.Params.noindex }}
  <meta name="robots" content="noindex, nofollow">
{{ end }}
```

## Secrets Management

### Required Secrets

| Secret | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM access key (or use OIDC) |
| `AWS_SECRET_ACCESS_KEY` | IAM secret key (or use OIDC) |
| `S3_BUCKET` | S3 bucket name |
| `CLOUDFRONT_DISTRIBUTION_ID` | CloudFront distribution ID |
| `PROD_S3_BUCKET` | Production bucket (multi-env) |
| `STAGING_S3_BUCKET` | Staging bucket (multi-env) |

### Least-Privilege IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3Deploy",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::example-com-hugo",
        "arn:aws:s3:::example-com-hugo/*"
      ]
    },
    {
      "Sid": "CloudFrontInvalidate",
      "Effect": "Allow",
      "Action": [
        "cloudfront:CreateInvalidation"
      ],
      "Resource": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
    }
  ]
}
```

## Preview Deployments

### Deploy PR Previews

```yaml
name: PR Preview

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build Preview
        run: |
          hugo --minify --gc \
            --baseURL "https://preview-pr${{ github.event.pull_request.number }}.example.com/" \
            --environment staging

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Deploy Preview
        run: |
          aws s3 sync public/ \
            s3://preview-example-com/pr-${{ github.event.pull_request.number }}/ \
            --delete

      - name: Comment PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '🚀 Preview deployed: https://preview-pr${{ github.event.pull_request.number }}.example.com/'
            })
```

## Best Practices

**DO:**
- Use `fetch-depth: 0` for Git history (needed for `.GitInfo`)
- Use `submodules: recursive` for Hugo themes
- Set proper cache headers per content type
- Invalidate CloudFront after deployment
- Use OIDC instead of long-lived access keys
- Run validation (HTML check, link check) before deploy
- Use environment-specific Hugo config
- Add noindex to staging environments

**DON'T:**
- Store AWS credentials as plaintext
- Skip CloudFront invalidation
- Deploy without building with `--minify`
- Use the same S3 bucket for staging and production
- Skip `--delete` flag (leaves orphaned files)
- Deploy PRs to production infrastructure
- Forget `--gc` and `--cleanDestinationDir` flags

## Guidelines

### Essential

- Automated build on push to main
- Hugo build with `--minify`
- S3 sync with `--delete`
- CloudFront cache invalidation
- AWS credentials in CI secrets

### Recommended

- Multi-environment (staging + production)
- HTML/link validation
- Smoke test after deployment
- OIDC authentication
- Deployment script for cache headers
- Hugo native deploy with matchers

### Advanced

- PR preview deployments
- Lighthouse CI integration
- Rollback automation
- Build caching
- Deployment notifications (Slack/email)
- Parallel builds for multi-language sites

## Benefits

Automated. Every push triggers build and deploy.

Consistent. Same build process every time.

Fast. Hugo builds in seconds, S3 sync is incremental.

Safe. Staging environment catches issues before production.

Auditable. Every deployment tied to a Git commit.

## Related

- [hugo-s3-deployment.md](./hugo-s3-deployment.md) - S3 bucket setup
- [hugo-cloudfront-integration.md](./hugo-cloudfront-integration.md) - CloudFront configuration
- [hugo-lambda-edge-rewrites.md](./hugo-lambda-edge-rewrites.md) - URL rewriting at the edge
