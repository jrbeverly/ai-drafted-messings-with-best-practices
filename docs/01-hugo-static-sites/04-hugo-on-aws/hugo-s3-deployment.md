# Hugo S3 Deployment

Deploy Hugo static sites to Amazon S3. Static website hosting. S3 bucket configuration. Hugo publish workflow.

## Principle

Deploy Hugo sites to S3 for cost-effective, scalable static hosting. Use S3 static website hosting for simple deployments or pair with CloudFront for production-grade delivery. Automate deployments with Hugo's built-in deploy command or AWS CLI.

## S3 Static Website Hosting

### How It Works

**S3 serves static files directly over HTTP:**

```
User Request → S3 Bucket (Static Website Hosting) → HTML/CSS/JS Response
```

**With CloudFront (recommended for production):**

```
User Request → CloudFront Edge → S3 Origin → Cached Response
```

### Why S3 for Hugo?

**Cost:**
- Storage: ~$0.023/GB/month
- Requests: ~$0.0004/1000 GET requests
- Typical Hugo site (50MB): ~$0.01/month + request costs
- Zero cost when idle (no servers running)

**Scalability:**
- Handles any traffic level
- No capacity planning
- Built-in redundancy (99.999999999% durability)

**Simplicity:**
- No servers to manage
- No runtime to maintain
- Upload files and serve

## S3 Bucket Setup

### Create Bucket for Static Hosting

**AWS CLI:**

```bash
# Create bucket
aws s3 mb s3://example-com-hugo --region us-east-1

# Enable static website hosting
aws s3 website s3://example-com-hugo \
  --index-document index.html \
  --error-document 404.html
```

### Bucket Policy (Public Access)

**For direct S3 hosting (without CloudFront):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::example-com-hugo/*"
    }
  ]
}
```

**Apply policy:**

```bash
aws s3api put-bucket-policy \
  --bucket example-com-hugo \
  --policy file://bucket-policy.json
```

### Block Public Access Settings

**For direct S3 hosting, disable block public access:**

```bash
aws s3api put-public-access-block \
  --bucket example-com-hugo \
  --public-access-block-configuration \
    BlockPublicAcls=false,\
    IgnorePublicAcls=false,\
    BlockPublicPolicy=false,\
    RestrictPublicBuckets=false
```

**For CloudFront origin (recommended), keep public access blocked:**

```bash
# Keep defaults - block all public access
# CloudFront uses Origin Access Control (OAC) instead
aws s3api put-public-access-block \
  --bucket example-com-hugo \
  --public-access-block-configuration \
    BlockPublicAcls=true,\
    IgnorePublicAcls=true,\
    BlockPublicPolicy=true,\
    RestrictPublicBuckets=true
```

### Bucket Policy for CloudFront OAC

**Allow only CloudFront to access S3:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::example-com-hugo/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
        }
      }
    }
  ]
}
```

## Hugo Build for S3

### Build Command

```bash
# Production build
hugo --minify --gc --cleanDestinationDir

# Build with specific base URL
hugo --minify --baseURL "https://example.com/"

# Build to custom output directory
hugo --minify --destination ./deploy
```

### Hugo Configuration for S3

**config.toml:**

```toml
baseURL = "https://example.com/"
languageCode = "en-us"
title = "My Hugo Site"

[minify]
  disableHTML = false
  disableCSS = false
  disableJS = false
  disableJSON = false
  disableSVG = false
  disableXML = false

[outputs]
  home = ["HTML", "RSS", "JSON"]

# Asset fingerprinting for cache busting
[params]
  fingerprint = true
```

### Asset Fingerprinting

**layouts/partials/head/styles.html:**

```go-html-template
{{ $styles := resources.Get "css/main.css" }}
{{ $styles = $styles | minify | fingerprint "sha256" }}
<link rel="stylesheet" href="{{ $styles.RelPermalink }}" integrity="{{ $styles.Data.Integrity }}" crossorigin="anonymous">
```

**Why fingerprint?** Enables aggressive S3/CloudFront caching with automatic cache busting when content changes.

## Deployment Methods

### Method 1: AWS CLI Sync

**Basic sync:**

```bash
# Build and sync
hugo --minify
aws s3 sync public/ s3://example-com-hugo/ --delete
```

**Optimized sync with cache headers:**

```bash
#!/bin/bash
# deploy.sh - Hugo S3 deployment with proper cache headers

BUCKET="example-com-hugo"
DISTRIBUTION_ID="EDFDVBD6EXAMPLE"

# Build
hugo --minify --gc --cleanDestinationDir

# Sync HTML files (short cache, must revalidate)
aws s3 sync public/ "s3://${BUCKET}/" \
  --exclude "*" \
  --include "*.html" \
  --cache-control "public, max-age=300, must-revalidate" \
  --content-type "text/html; charset=utf-8" \
  --delete

# Sync fingerprinted assets (long cache)
aws s3 sync public/ "s3://${BUCKET}/" \
  --exclude "*" \
  --include "*.css" \
  --include "*.js" \
  --cache-control "public, max-age=31536000, immutable" \
  --delete

# Sync images (medium cache)
aws s3 sync public/ "s3://${BUCKET}/" \
  --exclude "*" \
  --include "*.jpg" \
  --include "*.jpeg" \
  --include "*.png" \
  --include "*.gif" \
  --include "*.svg" \
  --include "*.webp" \
  --include "*.avif" \
  --include "*.ico" \
  --cache-control "public, max-age=86400" \
  --delete

# Sync fonts (long cache)
aws s3 sync public/ "s3://${BUCKET}/" \
  --exclude "*" \
  --include "*.woff" \
  --include "*.woff2" \
  --include "*.ttf" \
  --include "*.eot" \
  --cache-control "public, max-age=31536000, immutable" \
  --delete

# Sync remaining files (XML, JSON, txt, etc.)
aws s3 sync public/ "s3://${BUCKET}/" \
  --exclude "*.html" \
  --exclude "*.css" \
  --exclude "*.js" \
  --exclude "*.jpg" \
  --exclude "*.jpeg" \
  --exclude "*.png" \
  --exclude "*.gif" \
  --exclude "*.svg" \
  --exclude "*.webp" \
  --exclude "*.avif" \
  --exclude "*.ico" \
  --exclude "*.woff" \
  --exclude "*.woff2" \
  --exclude "*.ttf" \
  --exclude "*.eot" \
  --cache-control "public, max-age=3600" \
  --delete

# Invalidate CloudFront cache (if using CloudFront)
aws cloudfront create-invalidation \
  --distribution-id "${DISTRIBUTION_ID}" \
  --paths "/*"

echo "Deployment complete!"
```

### Method 2: Hugo Deploy (Built-in)

**config.toml:**

```toml
[deployment]
  [[deployment.targets]]
    name = "production"
    URL = "s3://example-com-hugo?region=us-east-1"

  [[deployment.matchers]]
    # HTML files - short cache
    pattern = "^.+\\.html$"
    cacheControl = "public, max-age=300, must-revalidate"
    contentType = "text/html; charset=utf-8"
    gzip = true

  [[deployment.matchers]]
    # Fingerprinted CSS/JS - immutable cache
    pattern = "^.+\\.(css|js)$"
    cacheControl = "public, max-age=31536000, immutable"
    gzip = true

  [[deployment.matchers]]
    # Images
    pattern = "^.+\\.(png|jpg|jpeg|gif|svg|webp|avif)$"
    cacheControl = "public, max-age=86400"

  [[deployment.matchers]]
    # Fonts - immutable cache
    pattern = "^.+\\.(woff|woff2|ttf|eot)$"
    cacheControl = "public, max-age=31536000, immutable"

  [[deployment.matchers]]
    # XML, JSON, etc.
    pattern = "^.+\\.(xml|json|txt)$"
    cacheControl = "public, max-age=3600"
    gzip = true
```

**Deploy command:**

```bash
# Build and deploy
hugo --minify
hugo deploy --target production

# Deploy with confirmation
hugo deploy --target production --confirm

# Dry run (preview changes)
hugo deploy --target production --dryRun

# Deploy with max delete limit (safety)
hugo deploy --target production --maxDeletes 50
```

### Method 3: AWS CDK / Terraform

**See [hugo-cloudfront-integration.md](./hugo-cloudfront-integration.md) for Infrastructure as Code approaches.**

## Cache-Control Strategy

### Cache Tiers

| Content Type | max-age | Strategy |
|---|---|---|
| HTML | 300s (5 min) | Short cache, must-revalidate |
| Fingerprinted CSS/JS | 31536000 (1 year) | Immutable, cache bust via filename |
| Images | 86400 (1 day) | Medium cache |
| Fonts | 31536000 (1 year) | Immutable |
| XML/JSON (feeds, sitemap) | 3600 (1 hour) | Medium-short cache |
| robots.txt | 86400 (1 day) | Medium cache |

### Why These Values?

**HTML (5 minutes):**
- Content changes frequently
- Must revalidate ensures fresh content
- Short enough for updates, long enough to reduce requests

**Fingerprinted assets (1 year, immutable):**
- Filename changes when content changes (e.g., `main.abc123.css`)
- Safe to cache forever
- `immutable` tells browser not to revalidate

**Images (1 day):**
- Rarely change once published
- Balance between freshness and performance
- CloudFront can invalidate if needed

## Content Types

### Set Correct MIME Types

**Common issues with S3:**

```bash
# S3 may not set correct content types automatically
# Explicitly set for critical files

# HTML with UTF-8
aws s3 cp public/index.html s3://example-com-hugo/index.html \
  --content-type "text/html; charset=utf-8"

# CSS
aws s3 cp public/css/main.css s3://example-com-hugo/css/main.css \
  --content-type "text/css; charset=utf-8"

# JavaScript modules
aws s3 cp public/js/app.js s3://example-com-hugo/js/app.js \
  --content-type "application/javascript; charset=utf-8"

# SVG
aws s3 cp public/images/logo.svg s3://example-com-hugo/images/logo.svg \
  --content-type "image/svg+xml"

# Web fonts
aws s3 cp public/fonts/inter.woff2 s3://example-com-hugo/fonts/inter.woff2 \
  --content-type "font/woff2"
```

**Hugo deploy handles this automatically** when matchers include `contentType`.

## Error Pages

### Custom 404 Page

**Hugo content/404.md:**

```markdown
---
title: "Page Not Found"
layout: "404"
---
```

**layouts/404.html:**

```go-html-template
{{ define "main" }}
<div class="error-page">
  <h1>404</h1>
  <p>The page you're looking for doesn't exist.</p>
  <a href="{{ "/" | relURL }}">Go home</a>
</div>
{{ end }}
```

**S3 error document configuration:**

```bash
aws s3 website s3://example-com-hugo \
  --index-document index.html \
  --error-document 404.html
```

### S3 Website Hosting Limitations

**S3 static website hosting:**
- Only supports HTTP (not HTTPS directly)
- Custom domain requires Route 53 or DNS alias
- No custom headers
- Limited redirect rules

**Solution:** Use CloudFront in front of S3 for HTTPS, custom headers, and better routing.

## S3 Redirect Rules

### Hugo Pretty URLs

**Hugo generates clean URLs by default:**

```
content/blog/my-post.md → public/blog/my-post/index.html
```

**S3 handles this natively** with index document routing.

### Redirect Rules

**S3 website redirect rules (JSON):**

```json
[
  {
    "Condition": {
      "KeyPrefixEquals": "old-blog/"
    },
    "Redirect": {
      "ReplaceKeyPrefixWith": "blog/",
      "HttpRedirectCode": "301"
    }
  },
  {
    "Condition": {
      "HttpErrorCodeReturnedEquals": "404"
    },
    "Redirect": {
      "HostName": "example.com",
      "ReplaceKeyWith": "404.html",
      "HttpRedirectCode": "302"
    }
  }
]
```

**Apply redirect rules:**

```bash
aws s3api put-bucket-website \
  --bucket example-com-hugo \
  --website-configuration '{
    "IndexDocument": {"Suffix": "index.html"},
    "ErrorDocument": {"Key": "404.html"},
    "RoutingRules": [
      {
        "Condition": {"KeyPrefixEquals": "old-blog/"},
        "Redirect": {"ReplaceKeyPrefixWith": "blog/", "HttpRedirectCode": "301"}
      }
    ]
  }'
```

## Hugo Aliases (Redirects)

### Front Matter Aliases

**Hugo generates redirect HTML files for aliases:**

```yaml
---
title: "Hugo Performance Tips"
aliases:
  - /blog/old-hugo-tips/
  - /tutorials/hugo-performance/
---
```

**Generated redirect file (public/blog/old-hugo-tips/index.html):**

```html
<!DOCTYPE html>
<html>
<head>
  <link rel="canonical" href="https://example.com/blog/hugo-performance-tips/">
  <meta http-equiv="refresh" content="0; url=https://example.com/blog/hugo-performance-tips/">
</head>
</html>
```

**These work on S3** without any special configuration.

## IAM Permissions

### Deployment IAM Policy

**Minimum permissions for CI/CD deployment:**

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
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::example-com-hugo",
        "arn:aws:s3:::example-com-hugo/*"
      ]
    },
    {
      "Sid": "CloudFrontInvalidation",
      "Effect": "Allow",
      "Action": [
        "cloudfront:CreateInvalidation",
        "cloudfront:GetInvalidation"
      ],
      "Resource": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
    }
  ]
}
```

## Multi-Environment Setup

### Staging and Production Buckets

```bash
# Staging
hugo --minify --baseURL "https://staging.example.com/"
aws s3 sync public/ s3://staging-example-com-hugo/ --delete

# Production
hugo --minify --baseURL "https://example.com/"
aws s3 sync public/ s3://example-com-hugo/ --delete
```

### Hugo Deploy Multi-Target

**config.toml:**

```toml
[deployment]
  [[deployment.targets]]
    name = "staging"
    URL = "s3://staging-example-com-hugo?region=us-east-1"

  [[deployment.targets]]
    name = "production"
    URL = "s3://example-com-hugo?region=us-east-1"

  # Matchers apply to all targets
  [[deployment.matchers]]
    pattern = "^.+\\.html$"
    cacheControl = "public, max-age=300, must-revalidate"
    gzip = true
```

**Deploy to specific target:**

```bash
hugo deploy --target staging
hugo deploy --target production
```

## Testing

### Verify Deployment

```bash
# Check website endpoint (direct S3)
curl -I http://example-com-hugo.s3-website-us-east-1.amazonaws.com/

# Check specific file
curl -I http://example-com-hugo.s3-website-us-east-1.amazonaws.com/index.html

# Verify cache headers
curl -s -D- http://example-com-hugo.s3-website-us-east-1.amazonaws.com/css/main.css | grep -i cache-control

# List bucket contents
aws s3 ls s3://example-com-hugo/ --recursive --human-readable --summarize
```

### Common Issues

**403 Forbidden:**
- Check bucket policy allows public read (if direct hosting)
- Check CloudFront OAC is configured (if using CloudFront)
- Verify block public access settings

**404 Not Found:**
- Verify index document is set to `index.html`
- Check Hugo `baseURL` matches deployment URL
- Verify trailing slashes in URLs

**Stale content:**
- Clear CloudFront cache: `aws cloudfront create-invalidation`
- Check cache-control headers on files
- Verify `--delete` flag is used in sync

## Best Practices

**DO:**
- Use CloudFront in front of S3 for production
- Set proper cache-control headers per content type
- Use Hugo asset fingerprinting for cache busting
- Use `--delete` flag to remove old files
- Keep S3 bucket private, use CloudFront OAC
- Use IAM policies with least privilege
- Test deployments in staging first

**DON'T:**
- Make S3 bucket public when using CloudFront
- Use same cache headers for all file types
- Skip CloudFront invalidation after deploy
- Hardcode AWS credentials in scripts
- Deploy without `--minify` flag
- Forget to set `baseURL` for the target environment

## Guidelines

### Essential

- S3 bucket with static website hosting enabled
- Hugo `--minify` for production builds
- Correct `baseURL` per environment
- Custom 404 error page

### Recommended

- CloudFront distribution for HTTPS and caching
- Cache-control headers per content type
- Hugo asset fingerprinting
- CI/CD automated deployment
- Staging environment

### Advanced

- Multi-region S3 replication
- CloudFront Origin Access Control
- Lambda@Edge for URL rewriting
- Automated cache invalidation
- Infrastructure as Code (Terraform/CDK)

## Benefits

Zero-Cost-When-Idle. Pay only for storage and requests.

Global Scale. S3 + CloudFront serves content worldwide.

No Servers. No patching, no runtime, no maintenance.

Durability. S3 provides 99.999999999% data durability.

Simple Deployment. Upload files and serve.

## Related

- [hugo-cloudfront-integration.md](./hugo-cloudfront-integration.md) - CloudFront with Hugo
- [hugo-cicd-aws.md](./hugo-cicd-aws.md) - CI/CD pipeline for Hugo on AWS
- [hugo-lambda-edge-rewrites.md](./hugo-lambda-edge-rewrites.md) - URL rewriting with Lambda@Edge
