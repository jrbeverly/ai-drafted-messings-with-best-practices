# Hugo CloudFront Integration

CloudFront CDN with Hugo. Cache invalidation. Origin Access Control. HTTPS. Custom domains. Edge caching strategies.

## Principle

Use CloudFront as a CDN in front of S3 for Hugo sites. Serve content from edge locations worldwide with HTTPS, custom domains, and intelligent caching. CloudFront eliminates the limitations of direct S3 hosting while maintaining the zero-server architecture.

## Why CloudFront for Hugo?

### S3 Alone vs CloudFront + S3

| Feature          | S3 Only             | CloudFront + S3                    |
| ---------------- | ------------------- | ---------------------------------- |
| HTTPS            | No (HTTP only)      | Yes (free ACM certificate)         |
| Custom domain    | Limited (CNAME)     | Full support                       |
| Edge caching     | No                  | 400+ edge locations                |
| Custom headers   | No                  | Full control                       |
| HTTP/2, HTTP/3   | No                  | Yes                                |
| Compression      | No                  | Gzip + Brotli                      |
| URL rewriting    | Limited redirects   | Lambda@Edge / CloudFront Functions |
| Security headers | No                  | Response headers policy            |
| Cost             | Per-request from S3 | Cached, reduced S3 requests        |
| DDoS protection  | Basic               | AWS Shield Standard (free)         |

### Cost

**Typical Hugo site (1000 pages, 10K visitors/month):**

- S3 storage: ~$0.01/month
- CloudFront data transfer: ~$0.85/month (first 1TB free)
- CloudFront requests: ~$0.01/month
- ACM certificate: Free
- **Total: ~$1/month**

## CloudFront Distribution Setup

### Create Distribution (AWS CLI)

```bash
aws cloudfront create-distribution \
  --distribution-config '{
    "CallerReference": "hugo-site-2026",
    "Comment": "Hugo site CDN",
    "DefaultCacheBehavior": {
      "TargetOriginId": "S3-example-com-hugo",
      "ViewerProtocolPolicy": "redirect-to-https",
      "AllowedMethods": ["GET", "HEAD", "OPTIONS"],
      "CachedMethods": ["GET", "HEAD"],
      "Compress": true,
      "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
      "ResponseHeadersPolicyId": "67f7725c-6f97-4210-82d7-5512b31e9d03"
    },
    "Origins": {
      "Quantity": 1,
      "Items": [
        {
          "Id": "S3-example-com-hugo",
          "DomainName": "example-com-hugo.s3.us-east-1.amazonaws.com",
          "S3OriginConfig": {
            "OriginAccessIdentity": ""
          },
          "OriginAccessControlId": "E2QWRUHEXAMPLE"
        }
      ]
    },
    "Enabled": true,
    "DefaultRootObject": "index.html",
    "PriceClass": "PriceClass_100",
    "Aliases": {
      "Quantity": 1,
      "Items": ["example.com"]
    },
    "ViewerCertificate": {
      "ACMCertificateArn": "arn:aws:acm:us-east-1:123456789012:certificate/abc-123",
      "SSLSupportMethod": "sni-only",
      "MinimumProtocolVersion": "TLSv1.2_2021"
    },
    "HttpVersion": "http2and3"
  }'
```

### Origin Access Control (OAC)

**Create OAC (replaces deprecated Origin Access Identity):**

```bash
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "hugo-site-oac",
    "Description": "OAC for Hugo S3 bucket",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }'
```

**S3 bucket policy for OAC:**

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

## Terraform Configuration

### Complete Hugo CloudFront Setup

```hcl
# S3 bucket for Hugo site
resource "aws_s3_bucket" "hugo_site" {
  bucket = "example-com-hugo"
}

resource "aws_s3_bucket_versioning" "hugo_site" {
  bucket = aws_s3_bucket.hugo_site.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Block all public access (CloudFront handles access)
resource "aws_s3_bucket_public_access_block" "hugo_site" {
  bucket = aws_s3_bucket.hugo_site.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Origin Access Control
resource "aws_cloudfront_origin_access_control" "hugo_site" {
  name                              = "hugo-site-oac"
  description                       = "OAC for Hugo site S3 bucket"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# ACM Certificate (must be in us-east-1 for CloudFront)
resource "aws_acm_certificate" "hugo_site" {
  provider          = aws.us_east_1
  domain_name       = "example.com"
  subject_alternative_names = ["www.example.com"]
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

# CloudFront cache policy
resource "aws_cloudfront_cache_policy" "hugo_site" {
  name        = "hugo-site-cache-policy"
  comment     = "Cache policy for Hugo static site"
  default_ttl = 86400    # 1 day
  max_ttl     = 31536000 # 1 year
  min_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    cookies_config {
      cookie_behavior = "none"
    }
    headers_config {
      header_behavior = "none"
    }
    query_strings_config {
      query_string_behavior = "none"
    }
    enable_accept_encoding_brotli = true
    enable_accept_encoding_gzip   = true
  }
}

# Security headers policy
resource "aws_cloudfront_response_headers_policy" "hugo_site" {
  name    = "hugo-site-security-headers"
  comment = "Security headers for Hugo site"

  security_headers_config {
    strict_transport_security {
      access_control_max_age_sec = 31536000
      include_subdomains         = true
      preload                    = true
      override                   = true
    }

    content_type_options {
      override = true
    }

    frame_options {
      frame_option = "DENY"
      override     = true
    }

    xss_protection {
      mode_block = true
      protection = true
      override   = true
    }

    referrer_policy {
      referrer_policy = "strict-origin-when-cross-origin"
      override        = true
    }

    content_security_policy {
      content_security_policy = "default-src 'self'; img-src 'self' data: https:; script-src 'self'; style-src 'self' 'unsafe-inline'"
      override                = true
    }
  }

  custom_headers_config {
    items {
      header   = "Permissions-Policy"
      value    = "camera=(), microphone=(), geolocation=()"
      override = true
    }
  }
}

# CloudFront distribution
resource "aws_cloudfront_distribution" "hugo_site" {
  origin {
    domain_name              = aws_s3_bucket.hugo_site.bucket_regional_domain_name
    origin_id                = "S3-hugo-site"
    origin_access_control_id = aws_cloudfront_origin_access_control.hugo_site.id
  }

  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Hugo site CDN"
  default_root_object = "index.html"
  http_version        = "http2and3"
  price_class         = "PriceClass_100"

  aliases = ["example.com", "www.example.com"]

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-hugo-site"

    cache_policy_id            = aws_cloudfront_cache_policy.hugo_site.id
    response_headers_policy_id = aws_cloudfront_response_headers_policy.hugo_site.id

    viewer_protocol_policy = "redirect-to-https"
    compress               = true
  }

  # Custom error responses
  custom_error_response {
    error_code            = 403
    response_code         = 404
    response_page_path    = "/404.html"
    error_caching_min_ttl = 300
  }

  custom_error_response {
    error_code            = 404
    response_code         = 404
    response_page_path    = "/404.html"
    error_caching_min_ttl = 300
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.hugo_site.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  tags = {
    Project     = "hugo-site"
    Environment = "production"
  }
}

# S3 bucket policy allowing CloudFront OAC
resource "aws_s3_bucket_policy" "hugo_site" {
  bucket = aws_s3_bucket.hugo_site.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AllowCloudFrontServicePrincipal"
        Effect    = "Allow"
        Principal = { Service = "cloudfront.amazonaws.com" }
        Action    = "s3:GetObject"
        Resource  = "${aws_s3_bucket.hugo_site.arn}/*"
        Condition = {
          StringEquals = {
            "AWS:SourceArn" = aws_cloudfront_distribution.hugo_site.arn
          }
        }
      }
    ]
  })
}

# Route 53 records
resource "aws_route53_record" "hugo_site" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.hugo_site.domain_name
    zone_id                = aws_cloudfront_distribution.hugo_site.hosted_zone_id
    evaluate_target_health = false
  }
}

resource "aws_route53_record" "hugo_site_www" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "www.example.com"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.hugo_site.domain_name
    zone_id                = aws_cloudfront_distribution.hugo_site.hosted_zone_id
    evaluate_target_health = false
  }
}

# Outputs
output "cloudfront_domain" {
  value = aws_cloudfront_distribution.hugo_site.domain_name
}

output "cloudfront_distribution_id" {
  value = aws_cloudfront_distribution.hugo_site.id
}
```

## Cache Invalidation

### After Deployment

```bash
# Invalidate everything (use sparingly - first 1000/month free)
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*"

# Invalidate specific paths (more cost-effective)
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/index.html" "/blog/*" "/sitemap.xml"

# Check invalidation status
aws cloudfront get-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --id I2J0I21PCUYOIK
```

### Smart Invalidation Script

```bash
#!/bin/bash
# invalidate-changed.sh - Only invalidate changed files

BUCKET="example-com-hugo"
DISTRIBUTION_ID="EDFDVBD6EXAMPLE"
BUILD_DIR="public"

# Get list of changed files from hugo deploy
CHANGED_FILES=$(hugo deploy --target production --dryRun 2>&1 | \
  grep -E '^\+|^~' | \
  awk '{print "/" $2}')

if [ -z "$CHANGED_FILES" ]; then
  echo "No files changed, skipping invalidation"
  exit 0
fi

# Count changed files
NUM_CHANGED=$(echo "$CHANGED_FILES" | wc -l)

if [ "$NUM_CHANGED" -gt 50 ]; then
  # Too many files - invalidate everything
  echo "Invalidating all paths ($NUM_CHANGED files changed)"
  aws cloudfront create-invalidation \
    --distribution-id "$DISTRIBUTION_ID" \
    --paths "/*"
else
  # Invalidate specific paths
  echo "Invalidating $NUM_CHANGED paths"
  PATHS=$(echo "$CHANGED_FILES" | tr '\n' ' ')
  aws cloudfront create-invalidation \
    --distribution-id "$DISTRIBUTION_ID" \
    --paths $PATHS
fi
```

### Invalidation Costs

- First 1,000 invalidation paths/month: Free
- Additional paths: $0.005 per path
- Wildcard (`/*`) counts as 1 path
- Tip: Use `/*` for large deployments, specific paths for small updates

## Cache Behaviors

### Multiple Cache Behaviors

**Different caching for different content types:**

```hcl
# Static assets - long cache
resource "aws_cloudfront_distribution" "hugo_site" {
  # ... origin config ...

  # Default behavior (HTML)
  default_cache_behavior {
    target_origin_id       = "S3-hugo-site"
    viewer_protocol_policy = "redirect-to-https"
    cache_policy_id        = aws_cloudfront_cache_policy.short_cache.id
    compress               = true
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
  }

  # Fingerprinted assets (immutable)
  ordered_cache_behavior {
    path_pattern           = "/css/*"
    target_origin_id       = "S3-hugo-site"
    viewer_protocol_policy = "redirect-to-https"
    cache_policy_id        = aws_cloudfront_cache_policy.immutable_cache.id
    compress               = true
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
  }

  ordered_cache_behavior {
    path_pattern           = "/js/*"
    target_origin_id       = "S3-hugo-site"
    viewer_protocol_policy = "redirect-to-https"
    cache_policy_id        = aws_cloudfront_cache_policy.immutable_cache.id
    compress               = true
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
  }

  ordered_cache_behavior {
    path_pattern           = "/images/*"
    target_origin_id       = "S3-hugo-site"
    viewer_protocol_policy = "redirect-to-https"
    cache_policy_id        = aws_cloudfront_cache_policy.image_cache.id
    compress               = false  # Images are already compressed
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
  }

  ordered_cache_behavior {
    path_pattern           = "/fonts/*"
    target_origin_id       = "S3-hugo-site"
    viewer_protocol_policy = "redirect-to-https"
    cache_policy_id        = aws_cloudfront_cache_policy.immutable_cache.id
    compress               = false
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
  }
}

# Short cache for HTML (5 minutes)
resource "aws_cloudfront_cache_policy" "short_cache" {
  name        = "hugo-short-cache"
  default_ttl = 300
  max_ttl     = 3600
  min_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    cookies_config { cookie_behavior = "none" }
    headers_config { header_behavior = "none" }
    query_strings_config { query_string_behavior = "none" }
    enable_accept_encoding_brotli = true
    enable_accept_encoding_gzip   = true
  }
}

# Immutable cache for fingerprinted assets (1 year)
resource "aws_cloudfront_cache_policy" "immutable_cache" {
  name        = "hugo-immutable-cache"
  default_ttl = 31536000
  max_ttl     = 31536000
  min_ttl     = 31536000

  parameters_in_cache_key_and_forwarded_to_origin {
    cookies_config { cookie_behavior = "none" }
    headers_config { header_behavior = "none" }
    query_strings_config { query_string_behavior = "none" }
    enable_accept_encoding_brotli = true
    enable_accept_encoding_gzip   = true
  }
}

# Image cache (1 day)
resource "aws_cloudfront_cache_policy" "image_cache" {
  name        = "hugo-image-cache"
  default_ttl = 86400
  max_ttl     = 604800
  min_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    cookies_config { cookie_behavior = "none" }
    headers_config { header_behavior = "none" }
    query_strings_config { query_string_behavior = "none" }
    enable_accept_encoding_brotli = false
    enable_accept_encoding_gzip   = false
  }
}
```

## Security Headers

### Response Headers Policy

```hcl
resource "aws_cloudfront_response_headers_policy" "security" {
  name = "hugo-security-headers"

  security_headers_config {
    # HSTS - force HTTPS
    strict_transport_security {
      access_control_max_age_sec = 31536000
      include_subdomains         = true
      preload                    = true
      override                   = true
    }

    # Prevent MIME sniffing
    content_type_options {
      override = true
    }

    # Prevent clickjacking
    frame_options {
      frame_option = "DENY"
      override     = true
    }

    # XSS protection
    xss_protection {
      mode_block = true
      protection = true
      override   = true
    }

    # Referrer policy
    referrer_policy {
      referrer_policy = "strict-origin-when-cross-origin"
      override        = true
    }

    # Content Security Policy
    content_security_policy {
      content_security_policy = join("; ", [
        "default-src 'self'",
        "img-src 'self' data: https:",
        "script-src 'self'",
        "style-src 'self' 'unsafe-inline'",
        "font-src 'self'",
        "connect-src 'self'",
        "frame-ancestors 'none'",
        "base-uri 'self'",
        "form-action 'self'"
      ])
      override = true
    }
  }

  custom_headers_config {
    items {
      header   = "Permissions-Policy"
      value    = "camera=(), microphone=(), geolocation=(), interest-cohort=()"
      override = true
    }

    items {
      header   = "X-Content-Type-Options"
      value    = "nosniff"
      override = true
    }
  }
}
```

## Custom Domain + HTTPS

### ACM Certificate

```bash
# Request certificate (must be in us-east-1 for CloudFront)
aws acm request-certificate \
  --region us-east-1 \
  --domain-name example.com \
  --subject-alternative-names "*.example.com" \
  --validation-method DNS

# After adding DNS validation records:
aws acm describe-certificate \
  --region us-east-1 \
  --certificate-arn arn:aws:acm:us-east-1:123456789012:certificate/abc-123
```

### Route 53 DNS

```bash
# Create alias record pointing to CloudFront
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "example.com",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z2FDTNDATAQYW2",
            "DNSName": "d1234567890.cloudfront.net",
            "EvaluateTargetHealth": false
          }
        }
      }
    ]
  }'
```

## WWW Redirect

### CloudFront Function for WWW Redirect

```javascript
// www-redirect.js - CloudFront Function
function handler(event) {
  var request = event.request;
  var host = request.headers.host.value;

  // Redirect www to non-www
  if (host.startsWith("www.")) {
    var newUrl = "https://" + host.substring(4) + request.uri;
    return {
      statusCode: 301,
      statusDescription: "Moved Permanently",
      headers: {
        location: { value: newUrl },
      },
    };
  }

  return request;
}
```

**Associate with distribution:**

```hcl
resource "aws_cloudfront_function" "www_redirect" {
  name    = "www-redirect"
  runtime = "cloudfront-js-2.0"
  comment = "Redirect www to non-www"
  publish = true
  code    = file("${path.module}/functions/www-redirect.js")
}

# In distribution default_cache_behavior:
function_association {
  event_type   = "viewer-request"
  function_arn = aws_cloudfront_function.www_redirect.arn
}
```

## Price Classes

### CloudFront Price Classes

| Price Class    | Edge Locations                                | Cost    |
| -------------- | --------------------------------------------- | ------- |
| PriceClass_All | All regions                                   | Highest |
| PriceClass_200 | US, Canada, Europe, Asia, Middle East, Africa | Medium  |
| PriceClass_100 | US, Canada, Europe                            | Lowest  |

**Recommendation for solo developer:** Start with `PriceClass_100` (cheapest). Most visitors are likely in US/Europe. Upgrade if you have significant Asia/Pacific traffic.

## Testing

### Verify CloudFront

```bash
# Check CloudFront distribution
curl -I https://example.com/

# Verify headers
curl -s -D- https://example.com/ | head -30

# Check cache status (Hit/Miss)
curl -s -D- https://example.com/ 2>&1 | grep -i x-cache

# Verify compression
curl -s -D- -H "Accept-Encoding: br,gzip" https://example.com/ | grep -i content-encoding

# Check security headers
curl -s -D- https://example.com/ | grep -iE 'strict-transport|x-frame|x-content-type|x-xss|referrer-policy|content-security'

# Verify HTTPS redirect
curl -I http://example.com/
# Should return 301 to https://
```

## Best Practices

**DO:**

- Use Origin Access Control (OAC), not Origin Access Identity (OAI)
- Enable HTTP/2 and HTTP/3
- Enable Brotli and Gzip compression
- Set security response headers policy
- Use PriceClass_100 to reduce costs
- Configure custom error responses for 404
- Use ACM for free HTTPS certificates
- Invalidate only changed paths when possible

**DON'T:**

- Make S3 bucket publicly accessible
- Skip security headers
- Use wildcard invalidation for every deployment
- Forget to configure custom error responses
- Use Origin Access Identity (deprecated)
- Skip TLS 1.2 minimum version
- Forget DNS records for custom domain

## Guidelines

### Essential

- CloudFront distribution with S3 origin
- Origin Access Control (OAC)
- ACM certificate for HTTPS
- Custom domain via Route 53
- Default root object (index.html)
- Custom 404 error response

### Recommended

- Security response headers policy
- Multiple cache behaviors per content type
- HTTP/2 and HTTP/3 enabled
- Brotli compression
- Smart cache invalidation
- WWW to non-www redirect

### Advanced

- CloudFront Functions for URL rewriting
- Lambda@Edge for complex routing
- Origin Shield for reduced origin load
- Real-time logs with Kinesis
- WAF integration for security
- Infrastructure as Code (Terraform)

## Benefits

Global Performance. Content served from nearest edge location.

Free HTTPS. ACM certificates at no cost.

Security. HSTS, CSP, and other headers enforced at CDN level.

Cost-Effective. Caching reduces S3 request costs.

DDoS Protection. AWS Shield Standard included free.

## Related

- [hugo-s3-deployment.md](./hugo-s3-deployment.md) - S3 bucket setup and deployment
- [hugo-cicd-aws.md](./hugo-cicd-aws.md) - Automated deployment pipeline
- [hugo-lambda-edge-rewrites.md](./hugo-lambda-edge-rewrites.md) - URL rewriting at the edge
