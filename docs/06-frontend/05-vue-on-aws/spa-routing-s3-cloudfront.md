# SPA Routing on S3 and CloudFront

Solve the SPA routing problem where client-side routes return 404 errors on S3. Configure CloudFront to serve `index.html` for all routes while preserving real 404 detection.

**Keywords:** spa-routing, client-side-routing, s3-404, cloudfront-error-response, cloudfront-function, vue-router-history-mode, prerendering, seo, terraform, custom-error-pages

## Principle

Vue Router in history mode produces clean URLs like `/dashboard/users/123`. These routes exist only in the browser's JavaScript runtime. When a user refreshes the page or navigates directly to a deep link, the request hits S3, which has no file at that path and returns a 403 or 404 error. The solution is to intercept these errors and serve `index.html` instead, allowing Vue Router to handle the route client-side. CloudFront provides multiple mechanisms to accomplish this, each with different tradeoffs.

## The SPA Routing Problem

Understanding the failure mode:

```
1. User visits https://myapp.example.com/dashboard/users/123
2. CloudFront forwards request to S3 origin
3. S3 looks for object at key: dashboard/users/123
4. S3 finds no such object → returns 403 (private bucket) or 404 (public bucket)
5. User sees an error page instead of the Vue application

Expected behavior:
1. Any route should serve index.html
2. Vue Router reads the URL path
3. Vue Router renders the correct component
4. The application functions as if navigated client-side
```

This affects all SPAs using history mode routing (Vue Router, React Router, Angular Router). Hash mode (`/#/dashboard`) avoids this problem but produces ugly URLs and breaks SEO.

## Solution 1: S3 Error Document Redirect

The simplest approach. Configure S3 to serve `index.html` as the error document:

```hcl
# Only for direct S3 website hosting (dev environments)
resource "aws_s3_bucket_website_configuration" "spa" {
  bucket = aws_s3_bucket.spa.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "index.html"
  }
}
```

**Limitations:**
- Only works with S3 static website hosting endpoint (not S3 REST API)
- Returns HTTP 404 status code with `index.html` body (bad for SEO crawlers)
- Cannot distinguish between SPA routes and genuinely missing resources
- Not compatible with CloudFront OAC (which uses the REST API endpoint)

**Use case:** Development environments only, where simplicity matters more than correctness.

## Solution 2: CloudFront Custom Error Response

Configure CloudFront to intercept S3 error responses and return `index.html` with a 200 status:

```hcl
# env/my-app/cloudfront.tf

resource "aws_cloudfront_distribution" "spa" {
  # ... origin, default_cache_behavior, etc. ...

  # Intercept 403 from S3 (private bucket returns 403 for missing keys)
  custom_error_response {
    error_code            = 403
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 0
  }

  # Intercept 404 from S3 (in case bucket policy allows listing)
  custom_error_response {
    error_code            = 404
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 0
  }
}
```

How it works:

```
1. Request: /dashboard/users/123
2. CloudFront forwards to S3 origin
3. S3 returns 403 (object not found in private bucket)
4. CloudFront intercepts the 403
5. CloudFront fetches /index.html from S3 instead
6. CloudFront returns index.html with HTTP 200
7. Vue Router handles /dashboard/users/123 client-side
```

**Advantages:**
- Works with CloudFront OAC and private S3 buckets
- Simple to configure
- Returns correct 200 status for SPA routes

**Limitations:**
- Cannot distinguish SPA routes from genuinely missing API resources or assets
- All 403/404 errors become 200 responses (masks real errors)
- `error_caching_min_ttl = 0` is critical; without it, CloudFront caches the error response

**Use case:** Production environments where all requests to the CloudFront distribution are for the SPA. This is the most common and recommended approach.

## Solution 3: CloudFront Function for SPA Routing

The most flexible approach. A CloudFront Function inspects the request URI and rewrites paths that look like SPA routes to `/index.html`, while leaving asset requests untouched:

```javascript
// cloudfront-functions/spa-routing.js
// CloudFront Function: viewer-request event
// Rewrites SPA routes to /index.html while preserving asset requests

function handler(event) {
  var request = event.request;
  var uri = request.uri;

  // If the URI has a file extension, it is a real asset request — pass through
  if (uri.includes('.')) {
    return request;
  }

  // If the URI is exactly '/', serve index.html (already the default)
  if (uri === '/') {
    return request;
  }

  // All other paths are SPA routes — rewrite to /index.html
  request.uri = '/index.html';
  return request;
}
```

Deploy the CloudFront Function with Terraform:

```hcl
# env/my-app/cloudfront-functions.tf

resource "aws_cloudfront_function" "spa_routing" {
  name    = "${var.project_name}-${var.environment}-spa-routing"
  runtime = "cloudfront-js-2.0"
  comment = "Rewrites SPA routes to /index.html, passes asset requests through"
  publish = true

  code = file("${path.module}/functions/spa-routing.js")
}

resource "aws_cloudfront_distribution" "spa" {
  # ... origin configuration ...

  default_cache_behavior {
    # ... existing settings ...

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.spa_routing.arn
    }
  }
}
```

**Advantages:**
- Only rewrites paths that look like SPA routes (no file extension)
- Real asset 404s remain 404s (e.g., `/assets/missing-image.png` returns 404)
- No extra round-trip to S3 for SPA routes (rewrite happens at the edge)
- Sub-millisecond execution latency
- Cheaper than Lambda@Edge

**Limitations:**
- Logic is simple (CloudFront Functions have limited runtime: 10 KB code, 2 MB memory)
- Assumes SPA routes never contain a dot in the path segment
- Requires CloudFront Function quota (default 25 per distribution)

**Use case:** Production environments where you need to distinguish SPA routes from real asset requests, or where API and SPA share the same distribution.

## Handling Real 404s vs SPA Routes

When using Solution 3 (CloudFront Function), the function correctly routes asset requests through to S3. But the SPA itself must handle routes that do not match any Vue Router definition:

```ts
// router/routes.ts
import type { RouteRecordRaw } from 'vue-router'

export const routes: RouteRecordRaw[] = [
  {
    path: '/',
    name: 'home',
    component: () => import('@/views/HomeView.vue')
  },
  {
    path: '/dashboard',
    name: 'dashboard',
    component: () => import('@/views/DashboardView.vue'),
    meta: { requiresAuth: true }
  },
  {
    path: '/users/:userId',
    name: 'user-detail',
    component: () => import('@/views/UserDetailView.vue'),
    props: true
  },

  // Catch-all: must be the LAST route
  {
    path: '/:pathMatch(.*)*',
    name: 'not-found',
    component: () => import('@/views/NotFoundView.vue'),
    meta: { title: 'Page Not Found' }
  }
]
```

The not-found component should provide clear feedback:

```vue
<!-- views/NotFoundView.vue -->
<script setup lang="ts">
import { useRoute, useRouter } from 'vue-router'

const route = useRoute()
const router = useRouter()

const attemptedPath = route.fullPath
</script>

<template>
  <div class="not-found">
    <h1>404 - Page Not Found</h1>
    <p>
      The page <code>{{ attemptedPath }}</code> does not exist.
    </p>
    <div class="not-found-actions">
      <button @click="router.push({ name: 'home' })">Go to Home</button>
      <button @click="router.back()">Go Back</button>
    </div>
  </div>
</template>
```

The full flow with Solution 3:

```
Valid SPA route: /dashboard/users/123
  1. CloudFront Function sees no file extension → rewrites to /index.html
  2. S3 returns index.html (200)
  3. Vue Router matches /dashboard/users/123 → renders UserDetailView

Invalid SPA route: /nonexistent/page
  1. CloudFront Function sees no file extension → rewrites to /index.html
  2. S3 returns index.html (200)
  3. Vue Router finds no matching route → renders NotFoundView

Missing asset: /assets/missing-image.png
  1. CloudFront Function sees file extension → passes through
  2. S3 returns 403/404 (object does not exist)
  3. Browser receives real 404 error for the asset
```

## Prerendered Routes for SEO

For pages that must be crawlable by search engines, prerender them at build time so they exist as real HTML files on S3:

```bash
npm install -D vite-plugin-prerender
```

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { prerender } from 'vite-plugin-prerender'

export default defineConfig({
  plugins: [
    vue(),
    prerender({
      routes: ['/', '/about', '/pricing', '/blog'],
      // Renderer options
      renderer: {
        // Wait for Vue to finish rendering
        renderAfterDocumentEvent: 'app-rendered'
      }
    })
  ]
})
```

Trigger the render event in your app:

```ts
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

const app = createApp(App)
app.use(router)

router.isReady().then(() => {
  app.mount('#app')
  // Signal to prerenderer that the app is ready
  document.dispatchEvent(new Event('app-rendered'))
})
```

This produces actual HTML files at the prerendered paths:

```
dist/
├── index.html              # Prerendered home page
├── about/
│   └── index.html          # Prerendered /about
├── pricing/
│   └── index.html          # Prerendered /pricing
├── blog/
│   └── index.html          # Prerendered /blog
└── assets/
    └── ...                 # Hashed assets as usual
```

The CloudFront Function still works correctly because these paths resolve to real files on S3 via the `index_document` behavior.

## Complete Terraform Configuration

Full Terraform setup combining CloudFront, S3, and the SPA routing function:

```hcl
# env/my-app/main.tf

# --- S3 Bucket ---
resource "aws_s3_bucket" "spa" {
  bucket = "${var.project_name}-${var.environment}-spa"
}

resource "aws_s3_bucket_public_access_block" "spa" {
  bucket = aws_s3_bucket.spa.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# --- CloudFront OAC ---
resource "aws_cloudfront_origin_access_control" "spa" {
  name                              = "${var.project_name}-${var.environment}-spa-oac"
  description                       = "OAC for SPA S3 bucket"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# --- CloudFront Function ---
resource "aws_cloudfront_function" "spa_routing" {
  name    = "${var.project_name}-${var.environment}-spa-routing"
  runtime = "cloudfront-js-2.0"
  comment = "Rewrite SPA routes to /index.html"
  publish = true

  code = <<-EOF
    function handler(event) {
      var request = event.request;
      var uri = request.uri;
      if (uri.includes('.')) {
        return request;
      }
      if (uri === '/') {
        return request;
      }
      request.uri = '/index.html';
      return request;
    }
  EOF
}

# --- CloudFront Distribution ---
resource "aws_cloudfront_distribution" "spa" {
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  price_class         = "PriceClass_100"
  comment             = "${var.project_name} ${var.environment} SPA"

  origin {
    domain_name              = aws_s3_bucket.spa.bucket_regional_domain_name
    origin_id                = "s3-spa"
    origin_access_control_id = aws_cloudfront_origin_access_control.spa.id
  }

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-spa"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 86400
    max_ttl                = 31536000
    compress               = true

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.spa_routing.arn
    }
  }

  # Fallback error handling (belt and suspenders with the CloudFront Function)
  custom_error_response {
    error_code            = 403
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 0
  }

  custom_error_response {
    error_code            = 404
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 0
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }

  tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
  }
}

# --- S3 Bucket Policy ---
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

# --- Outputs ---
output "cloudfront_domain" {
  value = aws_cloudfront_distribution.spa.domain_name
}

output "s3_bucket_name" {
  value = aws_s3_bucket.spa.id
}

output "cloudfront_distribution_id" {
  value = aws_cloudfront_distribution.spa.id
}
```

## Best Practices

**DO:**
- Use CloudFront custom error responses as the default solution (Solution 2)
- Set `error_caching_min_ttl = 0` to prevent caching error responses
- Add a Vue Router catch-all route to display a proper 404 page in the SPA
- Use CloudFront Functions when you need to distinguish asset requests from SPA routes
- Prerender SEO-critical pages at build time
- Test deep link navigation after every deployment
- Combine Solution 2 and Solution 3 for defense in depth

**DON'T:**
- Use S3 website hosting in production (use CloudFront OAC for security)
- Cache error responses (leads to stale 200s for previously-missing assets that now exist)
- Assume all paths without extensions are SPA routes if your API shares the same distribution
- Forget the catch-all route in Vue Router (users would see a blank page instead of a 404)
- Use Lambda@Edge for simple SPA routing (CloudFront Functions are faster and cheaper)
- Hardcode `/index.html` as `response_page_path` without the leading slash

## Guidelines

**Essential:**
- Configure CloudFront to return `index.html` with 200 status for 403/404 errors from S3
- Define a catch-all route in Vue Router to handle unmatched paths
- Set `error_caching_min_ttl = 0` on all custom error responses
- Test direct URL access and page refresh on deployed SPA routes

**Recommended:**
- Use CloudFront Functions for SPA routing when sharing a distribution with an API
- Prerender landing pages, marketing pages, and blog posts for SEO
- Log CloudFront Function errors to diagnose routing issues
- Include both 403 and 404 custom error responses (S3 behavior varies by access configuration)

**Advanced:**
- Implement A/B testing by routing to different `index.html` versions via CloudFront Functions
- Use CloudFront Key Value Store with Functions for feature flags
- Build a routing function that supports both SPA routes and API path prefixes
- Monitor CloudFront Function metrics to detect routing anomalies

## Benefits

Clean URLs. Vue Router history mode produces human-readable, bookmarkable paths.

Deep link support. Users can share and bookmark any SPA route and it loads correctly.

SEO compatibility. Prerendered routes provide crawlable HTML for search engines.

Defense in depth. Combining CloudFront Functions with custom error responses covers edge cases.

Zero server management. Routing logic runs at the CDN edge with no origin server.

Correct HTTP semantics. SPA routes return 200; missing assets return 404.

## Related

- [vue-s3-deployment.md](./vue-s3-deployment.md) - Deploy Vue SPA to S3 with optimized caching
- [vue-cloudfront-optimization.md](./vue-cloudfront-optimization.md) - CloudFront cache strategies and security headers for Vue SPAs
