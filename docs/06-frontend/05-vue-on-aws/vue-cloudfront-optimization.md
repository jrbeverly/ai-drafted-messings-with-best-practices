# CloudFront Optimization for Vue SPAs

Configure CloudFront cache behaviors, security headers, compression, HTTP/2 and HTTP/3, custom domains with HTTPS, and performance monitoring for Vue single-page applications.

**Keywords:** cloudfront, cache-behavior, security-headers, csp, cors, gzip, brotli, http2, http3, acm-certificate, custom-domain, cloudfront-functions, response-headers-policy, terraform, spa-performance

## Principle

CloudFront is the delivery and security layer for a Vue SPA hosted on S3. Configure cache behaviors per content type so `index.html` is always fresh while fingerprinted assets are cached indefinitely. Inject security headers at the edge to harden the application without modifying source code. Enable compression and modern protocols to minimize transfer sizes and latency. Every configuration should be codified in Terraform for reproducibility across environments.

## Cache Behaviors per Content Type

A single CloudFront distribution can define multiple cache behaviors that match different path patterns. This allows distinct caching rules for the SPA entry point, hashed assets, and API proxying:

```hcl
# env/my-app/cloudfront.tf

resource "aws_cloudfront_distribution" "spa" {
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  http_version        = "http2and3"
  price_class         = "PriceClass_100"

  origin {
    domain_name              = aws_s3_bucket.spa.bucket_regional_domain_name
    origin_id                = "s3-spa"
    origin_access_control_id = aws_cloudfront_origin_access_control.spa.id
  }

  # Cache behavior for fingerprinted assets (long-lived cache)
  ordered_cache_behavior {
    path_pattern     = "/assets/*"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-spa"

    cache_policy_id          = aws_cloudfront_cache_policy.immutable_assets.id
    response_headers_policy_id = aws_cloudfront_response_headers_policy.security_headers.id

    viewer_protocol_policy = "redirect-to-https"
    compress               = true
  }

  # Default behavior (index.html and other non-fingerprinted files)
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-spa"

    cache_policy_id          = aws_cloudfront_cache_policy.no_cache_html.id
    response_headers_policy_id = aws_cloudfront_response_headers_policy.security_headers.id

    viewer_protocol_policy = "redirect-to-https"
    compress               = true

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.spa_routing.arn
    }
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate_validation.spa.certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
  }
}
```

### Cache Policies

Define managed cache policies for each content type:

```hcl
# Cache policy for fingerprinted assets: cache for 1 year
resource "aws_cloudfront_cache_policy" "immutable_assets" {
  name        = "${var.project_name}-${var.environment}-immutable-assets"
  comment     = "Long-lived cache for content-hashed assets"
  default_ttl = 31536000  # 365 days
  max_ttl     = 31536000
  min_ttl     = 31536000

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
    enable_accept_encoding_gzip   = true
    enable_accept_encoding_brotli = true
  }
}

# Cache policy for index.html: always revalidate
resource "aws_cloudfront_cache_policy" "no_cache_html" {
  name        = "${var.project_name}-${var.environment}-no-cache-html"
  comment     = "No caching for SPA entry point"
  default_ttl = 0
  max_ttl     = 0
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
    enable_accept_encoding_gzip   = true
    enable_accept_encoding_brotli = true
  }
}
```

Cache behavior summary:

| Path Pattern | Cache Policy | TTL | Compression | Rationale |
|---|---|---|---|---|
| `/assets/*` | Immutable | 1 year | Gzip + Brotli | Content-hashed filenames; safe to cache forever |
| `*` (default) | No cache | 0 | Gzip + Brotli | `index.html` must always be fresh |

## Security Headers for SPAs

Inject security headers at the CloudFront edge using a response headers policy. This avoids modifying application code and applies headers consistently to all responses:

```hcl
# env/my-app/security-headers.tf

resource "aws_cloudfront_response_headers_policy" "security_headers" {
  name    = "${var.project_name}-${var.environment}-security-headers"
  comment = "Security headers for Vue SPA"

  security_headers_config {
    # Prevent MIME type sniffing
    content_type_options {
      override = true
    }

    # Enforce HTTPS for 1 year, include subdomains
    strict_transport_security {
      access_control_max_age_sec = 31536000
      include_subdomains         = true
      preload                    = true
      override                   = true
    }

    # Prevent clickjacking
    frame_options {
      frame_option = "DENY"
      override     = true
    }

    # Control referrer information
    referrer_policy {
      referrer_policy = "strict-origin-when-cross-origin"
      override        = true
    }

    # Content Security Policy
    content_security_policy {
      content_security_policy = join("; ", [
        "default-src 'self'",
        "script-src 'self'",
        "style-src 'self' 'unsafe-inline'",
        "img-src 'self' data: https:",
        "font-src 'self'",
        "connect-src 'self' https://api.${var.domain_name}",
        "frame-ancestors 'none'",
        "base-uri 'self'",
        "form-action 'self'",
        "upgrade-insecure-requests"
      ])
      override = true
    }

    # Disable cross-origin embedding
    xss_protection {
      mode_block = true
      protection = true
      override   = true
    }
  }

  # CORS headers for API calls from the SPA
  cors_config {
    access_control_allow_origins {
      items = ["https://${var.domain_name}"]
    }
    access_control_allow_methods {
      items = ["GET", "HEAD", "OPTIONS"]
    }
    access_control_allow_headers {
      items = ["Authorization", "Content-Type", "X-Requested-With"]
    }
    access_control_max_age_sec = 86400
    origin_override            = true
  }

  # Custom headers
  custom_headers_config {
    items {
      header   = "Permissions-Policy"
      value    = "camera=(), microphone=(), geolocation=(), payment=()"
      override = true
    }
    items {
      header   = "Cross-Origin-Opener-Policy"
      value    = "same-origin"
      override = true
    }
    items {
      header   = "Cross-Origin-Resource-Policy"
      value    = "same-origin"
      override = true
    }
  }
}
```

### CSP for Vue SPAs with Vite

Vite injects inline scripts in development mode. In production, all scripts are external files. Avoid `'unsafe-inline'` for `script-src`:

```
# Production CSP (no inline scripts)
script-src 'self';

# If you must allow inline styles (some UI libraries require it)
style-src 'self' 'unsafe-inline';

# Allow API calls to your backend
connect-src 'self' https://api.myapp.example.com;

# Allow images from your CDN and data URIs
img-src 'self' data: https://cdn.myapp.example.com;
```

If the application uses a hash or nonce-based CSP, configure Vite to output the appropriate attributes:

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  build: {
    // Disable inline scripts to comply with strict CSP
    assetsInlineLimit: 0
  },
  // Generate SRI hashes for script tags
  experimental: {
    renderBuiltUrl(filename) {
      return { relative: true }
    }
  }
})
```

### CORS Configuration for API Calls

When the SPA makes API calls to a different domain (e.g., `api.myapp.example.com`), configure CORS on the API origin, not the SPA distribution. The SPA distribution only needs CORS if it serves assets consumed by other origins.

For the API distribution or API Gateway:

```hcl
# This belongs on the API side, not the SPA distribution
# Shown here for completeness

resource "aws_apigatewayv2_api" "api" {
  name          = "${var.project_name}-api"
  protocol_type = "HTTP"

  cors_configuration {
    allow_origins = [
      "https://${var.domain_name}",
      "https://staging.${var.domain_name}"
    ]
    allow_methods = ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
    allow_headers = ["Authorization", "Content-Type"]
    max_age       = 86400
  }
}
```

## Gzip and Brotli Compression

CloudFront automatically compresses responses when `compress = true` is set on the cache behavior and the viewer sends `Accept-Encoding: gzip` or `Accept-Encoding: br`. No additional configuration is needed:

```hcl
default_cache_behavior {
  # Enable automatic compression
  compress = true
  # ... other settings ...
}
```

Compression applies to these content types automatically:
- `text/html`, `text/css`, `text/javascript`, `application/javascript`
- `application/json`, `application/xml`
- `image/svg+xml`
- `font/woff`, `font/woff2` (already compressed, but CloudFront handles this)

For cache policies, enable both encoding types in the cache key so CloudFront stores separate compressed versions:

```hcl
parameters_in_cache_key_and_forwarded_to_origin {
  enable_accept_encoding_gzip   = true
  enable_accept_encoding_brotli = true
  # ... other settings ...
}
```

Typical compression ratios for Vue SPA assets:

| Asset Type | Uncompressed | Gzip | Brotli | Savings |
|---|---|---|---|---|
| JavaScript bundle | 250 KB | 75 KB | 62 KB | 75% |
| CSS bundle | 80 KB | 15 KB | 12 KB | 85% |
| HTML | 5 KB | 2 KB | 1.5 KB | 70% |

Brotli provides 15-20% better compression than gzip for text-based assets. CloudFront serves Brotli when the viewer supports it and gzip as fallback.

## HTTP/2 and HTTP/3 for SPAs

Enable HTTP/2 and HTTP/3 for optimal SPA delivery:

```hcl
resource "aws_cloudfront_distribution" "spa" {
  # Enable both HTTP/2 and HTTP/3 (QUIC)
  http_version = "http2and3"
  # ... other configuration ...
}
```

Why HTTP/2 and HTTP/3 matter for SPAs:

- **Multiplexing:** Multiple JS/CSS/image requests share a single connection. Vue SPAs with code splitting make many parallel requests on initial load.
- **Header compression (HPACK/QPACK):** Repeated headers across requests are compressed, reducing overhead for API calls.
- **Server push (HTTP/2):** Not commonly used with S3 origins, but available with custom origins.
- **0-RTT connection resumption (HTTP/3):** Returning visitors establish connections faster with QUIC.
- **Connection migration (HTTP/3):** Users switching networks (Wi-Fi to cellular) maintain their connection.

No application code changes are required. CloudFront negotiates the best protocol with each viewer automatically.

## Custom Domain and HTTPS

Set up a custom domain with an ACM certificate. ACM certificates for CloudFront must be in `us-east-1`:

```hcl
# env/my-app/dns.tf

# ACM certificate (must be in us-east-1 for CloudFront)
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

resource "aws_acm_certificate" "spa" {
  provider          = aws.us_east_1
  domain_name       = var.domain_name
  validation_method = "DNS"

  subject_alternative_names = [
    "*.${var.domain_name}"
  ]

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

# DNS validation record
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.spa.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      type   = dvo.resource_record_type
      record = dvo.resource_record_value
    }
  }

  zone_id = data.aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60

  allow_overwrite = true
}

resource "aws_acm_certificate_validation" "spa" {
  provider                = aws.us_east_1
  certificate_arn         = aws_acm_certificate.spa.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}

# Route53 alias record pointing to CloudFront
resource "aws_route53_record" "spa" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.spa.domain_name
    zone_id                = aws_cloudfront_distribution.spa.hosted_zone_id
    evaluate_target_health = false
  }
}

# IPv6 AAAA record
resource "aws_route53_record" "spa_ipv6" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "AAAA"

  alias {
    name                   = aws_cloudfront_distribution.spa.domain_name
    zone_id                = aws_cloudfront_distribution.spa.hosted_zone_id
    evaluate_target_health = false
  }
}
```

Attach the certificate to the CloudFront distribution:

```hcl
resource "aws_cloudfront_distribution" "spa" {
  aliases = [var.domain_name]

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate_validation.spa.certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  # ... rest of distribution config ...
}
```

## CloudFront Functions for Header Injection

Use CloudFront Functions to add dynamic headers that cannot be set via the static response headers policy:

```javascript
// cloudfront-functions/security-headers.js
// CloudFront Function: viewer-response event
// Adds dynamic security headers to every response

function handler(event) {
  var response = event.response;
  var headers = response.headers;

  // Add cache-control based on content type
  var uri = event.request.uri;
  if (uri === '/' || uri === '/index.html' || !uri.includes('.')) {
    headers['cache-control'] = { value: 'no-cache, no-store, must-revalidate' };
  } else if (uri.startsWith('/assets/')) {
    headers['cache-control'] = { value: 'public, max-age=31536000, immutable' };
  }

  // Add request ID for debugging
  var requestId = event.request.headers['x-amz-cf-id']
    ? event.request.headers['x-amz-cf-id'].value
    : 'unknown';
  headers['x-request-id'] = { value: requestId };

  // Prevent caching of sensitive pages
  if (uri.startsWith('/account') || uri.startsWith('/settings')) {
    headers['cache-control'] = { value: 'no-store' };
    headers['pragma'] = { value: 'no-cache' };
  }

  return response;
}
```

Deploy the viewer-response function:

```hcl
resource "aws_cloudfront_function" "response_headers" {
  name    = "${var.project_name}-${var.environment}-response-headers"
  runtime = "cloudfront-js-2.0"
  comment = "Add dynamic response headers"
  publish = true

  code = file("${path.module}/functions/security-headers.js")
}

resource "aws_cloudfront_distribution" "spa" {
  default_cache_behavior {
    # Viewer-request function for SPA routing
    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.spa_routing.arn
    }

    # Viewer-response function for dynamic headers
    function_association {
      event_type   = "viewer-response"
      function_arn = aws_cloudfront_function.response_headers.arn
    }
  }
}
```

## Monitoring SPA Performance

### CloudWatch Metrics

Monitor CloudFront distribution health:

```hcl
# env/my-app/monitoring.tf

# Alarm on high 4xx error rate
resource "aws_cloudwatch_metric_alarm" "spa_4xx_rate" {
  alarm_name          = "${var.project_name}-${var.environment}-spa-4xx-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "4xxErrorRate"
  namespace           = "AWS/CloudFront"
  period              = 300
  statistic           = "Average"
  threshold           = 5
  alarm_description   = "SPA 4xx error rate exceeds 5%"

  dimensions = {
    DistributionId = aws_cloudfront_distribution.spa.id
    Region         = "Global"
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

# Alarm on high 5xx error rate
resource "aws_cloudwatch_metric_alarm" "spa_5xx_rate" {
  alarm_name          = "${var.project_name}-${var.environment}-spa-5xx-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "5xxErrorRate"
  namespace           = "AWS/CloudFront"
  period              = 300
  statistic           = "Average"
  threshold           = 1
  alarm_description   = "SPA 5xx error rate exceeds 1%"

  dimensions = {
    DistributionId = aws_cloudfront_distribution.spa.id
    Region         = "Global"
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

# Monitor cache hit rate
resource "aws_cloudwatch_metric_alarm" "spa_cache_hit_rate" {
  alarm_name          = "${var.project_name}-${var.environment}-spa-low-cache-hit"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 6
  metric_name         = "CacheHitRate"
  namespace           = "AWS/CloudFront"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "SPA cache hit rate dropped below 80%"

  dimensions = {
    DistributionId = aws_cloudfront_distribution.spa.id
    Region         = "Global"
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

### Client-Side Performance Monitoring

Instrument the Vue application to report real user performance metrics:

```ts
// utils/performance.ts
export function reportWebVitals(): void {
  if (typeof window === 'undefined') return

  // Use the web-vitals library
  import('web-vitals').then(({ onCLS, onFID, onLCP, onFCP, onTTFB }) => {
    onCLS(sendMetric)
    onFID(sendMetric)
    onLCP(sendMetric)
    onFCP(sendMetric)
    onTTFB(sendMetric)
  })
}

function sendMetric(metric: { name: string; value: number; id: string }): void {
  // Send to your analytics or monitoring endpoint
  const body = JSON.stringify({
    name: metric.name,
    value: Math.round(metric.name === 'CLS' ? metric.value * 1000 : metric.value),
    id: metric.id,
    page: window.location.pathname,
    timestamp: Date.now()
  })

  // Use sendBeacon to avoid blocking navigation
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/metrics', body)
  }
}
```

Initialize in the app entry point:

```ts
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'
import { reportWebVitals } from '@/utils/performance'

const app = createApp(App)
app.use(router)
app.mount('#app')

// Report performance metrics in production
if (import.meta.env.PROD) {
  reportWebVitals()
}
```

### CloudFront Function Monitoring

Monitor CloudFront Function execution:

```hcl
# Alarm on function errors
resource "aws_cloudwatch_metric_alarm" "function_errors" {
  alarm_name          = "${var.project_name}-${var.environment}-cf-function-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "FunctionExecutionErrors"
  namespace           = "AWS/CloudFront"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "CloudFront Function execution errors detected"

  dimensions = {
    DistributionId = aws_cloudfront_distribution.spa.id
    FunctionName   = aws_cloudfront_function.spa_routing.name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

## Complete Terraform Configuration

Full end-to-end Terraform module for a production Vue SPA on CloudFront:

```hcl
# env/my-app/variables.tf

variable "project_name" {
  description = "Project name used in resource naming"
  type        = string
}

variable "environment" {
  description = "Deployment environment"
  type        = string
}

variable "domain_name" {
  description = "Custom domain name for the SPA"
  type        = string
}

variable "route53_zone_id" {
  description = "Route53 hosted zone ID"
  type        = string
}

variable "api_domain_name" {
  description = "API domain for CSP connect-src"
  type        = string
}
```

```hcl
# env/my-app/outputs.tf

output "cloudfront_distribution_id" {
  description = "CloudFront distribution ID for cache invalidation"
  value       = aws_cloudfront_distribution.spa.id
}

output "cloudfront_domain_name" {
  description = "CloudFront distribution domain name"
  value       = aws_cloudfront_distribution.spa.domain_name
}

output "s3_bucket_name" {
  description = "S3 bucket name for deployment uploads"
  value       = aws_s3_bucket.spa.id
}

output "s3_bucket_arn" {
  description = "S3 bucket ARN for IAM policies"
  value       = aws_s3_bucket.spa.arn
}

output "spa_url" {
  description = "Full URL of the deployed SPA"
  value       = "https://${var.domain_name}"
}
```

## Best Practices

**DO:**
- Use separate cache behaviors for `/assets/*` (immutable) and default (no-cache)
- Enable both gzip and brotli compression on all cache behaviors
- Set `http_version = "http2and3"` for optimal delivery
- Use response headers policies for security headers instead of CloudFront Functions when possible
- Require TLS 1.2+ with `minimum_protocol_version = "TLSv1.2_2021"`
- Use SNI-only for the viewer certificate (cheaper, supported by all modern browsers)
- Monitor cache hit rate and error rates with CloudWatch alarms
- Use `PriceClass_100` to minimize costs (covers North America and Europe)

**DON'T:**
- Set long cache TTL on the default cache behavior (this caches `index.html`)
- Use `'unsafe-eval'` in CSP for production builds (Vite does not require it)
- Skip HSTS headers (leaves users vulnerable to downgrade attacks)
- Use `PriceClass_All` unless you have significant traffic in South America, Asia, or Africa
- Place the ACM certificate in a region other than `us-east-1` (CloudFront requires it there)
- Ignore CloudWatch alarms for 4xx/5xx spikes after deployments
- Disable IPv6 (no cost and improves availability on IPv6-only networks)
- Override security headers from the origin (set `override = true` on the policy)

## Guidelines

**Essential:**
- Create separate cache policies for fingerprinted assets and HTML entry point
- Enable compression on all cache behaviors
- Inject security headers via a CloudFront response headers policy
- Use ACM certificate with DNS validation for custom domains
- Enable HTTP/2 and HTTP/3

**Recommended:**
- Set up CloudWatch alarms for error rates and cache hit rate
- Use `Permissions-Policy` to disable unused browser APIs
- Report Web Vitals from the client for real user monitoring
- Include CORS configuration on the API origin, not the SPA distribution
- Use `PriceClass_100` to reduce costs while covering primary regions

**Advanced:**
- Implement CloudFront Function for dynamic cache-control headers per route
- Use CloudFront real-time logs with Kinesis for detailed access analytics
- Set up WAF rules on the CloudFront distribution for bot protection
- Enable CloudFront origin failover with an S3 replication bucket
- Use CloudFront Key Value Store for feature flags and configuration at the edge

## Benefits

Fast global delivery. CloudFront edge locations serve cached assets from the nearest point of presence.

Optimal caching. Separate cache behaviors ensure assets are cached aggressively while HTML stays fresh.

Strong security posture. Security headers enforced at the CDN edge without application code changes.

Reduced transfer costs. Brotli compression reduces asset sizes by 75-85%.

Modern protocol support. HTTP/2 multiplexing and HTTP/3 QUIC improve load performance without code changes.

Zero-cost certificate management. ACM provides free, auto-renewing TLS certificates.

Observable. CloudWatch metrics and alarms provide visibility into distribution health and performance.

## Related

- [vue-s3-deployment.md](./vue-s3-deployment.md) - Deploy Vue SPA to S3 with optimized caching
- [spa-routing-s3-cloudfront.md](./spa-routing-s3-cloudfront.md) - Handle SPA client-side routing on S3 and CloudFront
