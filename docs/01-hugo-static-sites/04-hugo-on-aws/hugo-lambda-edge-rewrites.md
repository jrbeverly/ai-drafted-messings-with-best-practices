# Hugo Lambda@Edge Rewrites

URL rewriting with Lambda@Edge. CloudFront Functions. Pretty URLs. Trailing slash normalization. Custom routing for Hugo on S3.

## Principle

Use Lambda@Edge or CloudFront Functions to handle URL rewriting for Hugo sites on S3/CloudFront. Solve the trailing slash problem, implement redirects, add security headers dynamically, and handle routing patterns that S3 static hosting cannot.

## The Hugo URL Problem on S3

### How Hugo Generates URLs

**Hugo produces clean URLs:**

```
content/blog/my-post.md → public/blog/my-post/index.html
```

**The URL structure:**

```
https://example.com/blog/my-post/     → S3 key: blog/my-post/index.html ✓
https://example.com/blog/my-post      → S3 key: blog/my-post (404!) ✗
```

### S3 Behavior

**S3 Static Website Hosting:**
- Appends `index.html` to directory-like requests (`/blog/` → `/blog/index.html`)
- Does NOT handle missing trailing slashes (`/blog` → 404)

**S3 as CloudFront Origin (REST API endpoint):**
- Does NOT append `index.html` at all
- `/blog/my-post/` → 404 (S3 REST API doesn't resolve directories)
- Requires explicit rewriting

### Solution: Edge Functions

**CloudFront Functions or Lambda@Edge** intercept requests and rewrite URLs before they reach S3.

## CloudFront Functions vs Lambda@Edge

| Feature | CloudFront Functions | Lambda@Edge |
|---|---|---|
| Runtime | JavaScript (ECMAScript 5.1) | Node.js, Python |
| Max execution time | 1ms | 5s (viewer) / 30s (origin) |
| Max memory | 2MB | 128MB (viewer) / 10GB (origin) |
| Network access | No | Yes |
| Geolocation headers | Yes | Yes |
| Request body access | No | Yes |
| Price | $0.10/million | $0.60/million |
| Event types | Viewer request, Viewer response | All 4 CloudFront events |

**Recommendation:** Use CloudFront Functions for URL rewriting (simpler, cheaper, faster). Use Lambda@Edge only when you need network access, longer execution, or origin-level manipulation.

## CloudFront Functions

### Trailing Slash + Index.html Rewrite

**The most common Hugo fix:**

```javascript
// url-rewrite.js - CloudFront Function (cloudfront-js-2.0)
function handler(event) {
  var request = event.request;
  var uri = request.uri;

  // If URI ends with '/', append index.html
  if (uri.endsWith('/')) {
    request.uri += 'index.html';
  }
  // If URI doesn't have a file extension, add trailing slash + index.html
  else if (!uri.includes('.')) {
    request.uri += '/index.html';
  }

  return request;
}
```

**This handles:**
- `/blog/my-post/` → `/blog/my-post/index.html`
- `/blog/my-post` → `/blog/my-post/index.html`
- `/about` → `/about/index.html`
- `/` → `/index.html`
- `/css/main.css` → `/css/main.css` (unchanged, has extension)

### Deploy CloudFront Function

**AWS CLI:**

```bash
# Create function
aws cloudfront create-function \
  --name hugo-url-rewrite \
  --function-config '{
    "Comment": "Rewrite URLs for Hugo pretty URLs",
    "Runtime": "cloudfront-js-2.0"
  }' \
  --function-code fileb://url-rewrite.js

# Test function
aws cloudfront test-function \
  --name hugo-url-rewrite \
  --if-match ETVPDKIKX0DER \
  --event-object '{
    "version": "1.0",
    "context": {"eventType": "viewer-request"},
    "viewer": {"ip": "1.2.3.4"},
    "request": {
      "method": "GET",
      "uri": "/blog/my-post",
      "headers": {},
      "querystring": {}
    }
  }'

# Publish function
aws cloudfront publish-function \
  --name hugo-url-rewrite \
  --if-match ETVPDKIKX0DER

# Associate with distribution
aws cloudfront update-distribution \
  --id EDFDVBD6EXAMPLE \
  --default-cache-behavior '{
    "FunctionAssociations": {
      "Quantity": 1,
      "Items": [
        {
          "FunctionARN": "arn:aws:cloudfront::123456789012:function/hugo-url-rewrite",
          "EventType": "viewer-request"
        }
      ]
    }
  }'
```

### Terraform Configuration

```hcl
resource "aws_cloudfront_function" "url_rewrite" {
  name    = "hugo-url-rewrite"
  runtime = "cloudfront-js-2.0"
  comment = "Rewrite URLs for Hugo pretty URLs"
  publish = true

  code = <<-EOF
    function handler(event) {
      var request = event.request;
      var uri = request.uri;

      if (uri.endsWith('/')) {
        request.uri += 'index.html';
      } else if (!uri.includes('.')) {
        request.uri += '/index.html';
      }

      return request;
    }
  EOF
}

resource "aws_cloudfront_distribution" "hugo_site" {
  # ... other config ...

  default_cache_behavior {
    # ... other settings ...

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.url_rewrite.arn
    }
  }
}
```

## Advanced CloudFront Functions

### Trailing Slash Redirect (SEO)

**Redirect non-trailing-slash URLs to trailing-slash (canonical):**

```javascript
// trailing-slash-redirect.js
function handler(event) {
  var request = event.request;
  var uri = request.uri;

  // Skip files with extensions
  if (uri.includes('.')) {
    return request;
  }

  // Root is fine
  if (uri === '/') {
    request.uri = '/index.html';
    return request;
  }

  // Redirect non-trailing-slash to trailing-slash (301)
  if (!uri.endsWith('/')) {
    var querystring = '';
    var qs = request.querystring;
    for (var key in qs) {
      if (querystring) querystring += '&';
      querystring += key + '=' + qs[key].value;
    }

    return {
      statusCode: 301,
      statusDescription: 'Moved Permanently',
      headers: {
        'location': {
          value: uri + '/' + (querystring ? '?' + querystring : '')
        }
      }
    };
  }

  // Append index.html
  request.uri += 'index.html';
  return request;
}
```

### WWW to Non-WWW Redirect

```javascript
// www-redirect.js
function handler(event) {
  var request = event.request;
  var host = request.headers.host.value;

  if (host.startsWith('www.')) {
    var newHost = host.substring(4);
    return {
      statusCode: 301,
      statusDescription: 'Moved Permanently',
      headers: {
        'location': {
          value: 'https://' + newHost + request.uri
        }
      }
    };
  }

  // Also handle URL rewriting
  var uri = request.uri;
  if (uri.endsWith('/')) {
    request.uri += 'index.html';
  } else if (!uri.includes('.')) {
    request.uri += '/index.html';
  }

  return request;
}
```

### Combined Function (All Hugo Rewrites)

```javascript
// hugo-rewrite-all.js - Complete Hugo URL handling
function handler(event) {
  var request = event.request;
  var uri = request.uri;
  var host = request.headers.host.value;

  // 1. WWW redirect
  if (host.startsWith('www.')) {
    return {
      statusCode: 301,
      statusDescription: 'Moved Permanently',
      headers: {
        'location': { value: 'https://' + host.substring(4) + uri }
      }
    };
  }

  // 2. Skip files with extensions (CSS, JS, images, etc.)
  if (uri.includes('.')) {
    return request;
  }

  // 3. Root path
  if (uri === '/') {
    request.uri = '/index.html';
    return request;
  }

  // 4. Ensure trailing slash (301 redirect)
  if (!uri.endsWith('/')) {
    return {
      statusCode: 301,
      statusDescription: 'Moved Permanently',
      headers: {
        'location': { value: uri + '/' }
      }
    };
  }

  // 5. Append index.html for directory URLs
  request.uri += 'index.html';
  return request;
}
```

### Custom Redirects Map

```javascript
// redirects.js - Handle Hugo alias-like redirects at edge
var redirects = {
  '/old-blog/': '/blog/',
  '/tutorials/hugo-tips/': '/blog/hugo-performance-tips/',
  '/about-us/': '/about/',
  '/feed/': '/index.xml',
  '/rss/': '/index.xml'
};

function handler(event) {
  var request = event.request;
  var uri = request.uri;

  // Check redirect map
  if (redirects[uri]) {
    return {
      statusCode: 301,
      statusDescription: 'Moved Permanently',
      headers: {
        'location': { value: redirects[uri] }
      }
    };
  }

  // Normal URL rewriting
  if (uri.endsWith('/')) {
    request.uri += 'index.html';
  } else if (!uri.includes('.')) {
    request.uri += '/index.html';
  }

  return request;
}
```

## Lambda@Edge

### When to Use Lambda@Edge

**Use Lambda@Edge instead of CloudFront Functions when you need:**
- Network access (fetch data from DynamoDB, API)
- Request body manipulation
- Complex logic (> 1ms execution)
- Origin request/response manipulation
- Larger code (> 10KB)

### Origin Request Rewrite

**Lambda@Edge for origin-request event:**

```javascript
// origin-request-rewrite.mjs (Node.js 20.x)
export const handler = async (event) => {
  const request = event.Records[0].cf.request;
  const uri = request.uri;

  // Append index.html for directory requests
  if (uri.endsWith('/')) {
    request.uri += 'index.html';
  } else if (!uri.includes('.')) {
    request.uri += '/index.html';
  }

  return request;
};
```

### A/B Testing with Lambda@Edge

```javascript
// ab-test.mjs - Route users to different versions
export const handler = async (event) => {
  const request = event.Records[0].cf.request;

  // Check for existing cookie
  const cookies = request.headers.cookie || [];
  let variant = null;

  for (const cookie of cookies) {
    const match = cookie.value.match(/ab-variant=([AB])/);
    if (match) {
      variant = match[1];
      break;
    }
  }

  // Assign variant if none
  if (!variant) {
    variant = Math.random() < 0.5 ? 'A' : 'B';
  }

  // Rewrite to variant-specific content
  if (variant === 'B' && request.uri.startsWith('/landing/')) {
    request.uri = request.uri.replace('/landing/', '/landing-b/');
  }

  return request;
};
```

### Geolocation Redirect

```javascript
// geo-redirect.mjs - Redirect based on country
export const handler = async (event) => {
  const request = event.Records[0].cf.request;
  const country = request.headers['cloudfront-viewer-country']?.[0]?.value;

  // Redirect to country-specific subdirectory
  const countryPaths = {
    'DE': '/de',
    'FR': '/fr',
    'ES': '/es',
    'JP': '/ja'
  };

  if (countryPaths[country] && request.uri === '/') {
    return {
      status: '302',
      statusDescription: 'Found',
      headers: {
        location: [{
          key: 'Location',
          value: countryPaths[country] + '/'
        }]
      }
    };
  }

  // Normal URL rewrite
  if (request.uri.endsWith('/')) {
    request.uri += 'index.html';
  } else if (!request.uri.includes('.')) {
    request.uri += '/index.html';
  }

  return request;
};
```

### Deploy Lambda@Edge

```hcl
# Lambda@Edge must be deployed in us-east-1
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

resource "aws_lambda_function" "edge_rewrite" {
  provider      = aws.us_east_1
  function_name = "hugo-edge-rewrite"
  role          = aws_iam_role.lambda_edge.arn
  handler       = "index.handler"
  runtime       = "nodejs20.x"
  publish       = true  # Required for Lambda@Edge

  filename         = data.archive_file.edge_rewrite.output_path
  source_code_hash = data.archive_file.edge_rewrite.output_base64sha256
}

resource "aws_iam_role" "lambda_edge" {
  name = "hugo-lambda-edge-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = [
            "lambda.amazonaws.com",
            "edgelambda.amazonaws.com"
          ]
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_edge_basic" {
  role       = aws_iam_role.lambda_edge.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# Associate with CloudFront distribution
resource "aws_cloudfront_distribution" "hugo_site" {
  default_cache_behavior {
    lambda_function_association {
      event_type   = "origin-request"
      lambda_arn   = aws_lambda_function.edge_rewrite.qualified_arn
      include_body = false
    }
  }
}
```

## Security Headers via Response

### CloudFront Function (Viewer Response)

```javascript
// security-headers.js - Add security headers
function handler(event) {
  var response = event.response;
  var headers = response.headers;

  headers['strict-transport-security'] = {
    value: 'max-age=31536000; includeSubdomains; preload'
  };
  headers['x-content-type-options'] = {
    value: 'nosniff'
  };
  headers['x-frame-options'] = {
    value: 'DENY'
  };
  headers['x-xss-protection'] = {
    value: '1; mode=block'
  };
  headers['referrer-policy'] = {
    value: 'strict-origin-when-cross-origin'
  };
  headers['permissions-policy'] = {
    value: 'camera=(), microphone=(), geolocation=()'
  };

  return response;
}
```

**Note:** CloudFront Response Headers Policies are preferred over functions for static headers. Use functions only for dynamic header logic.

## Testing

### Test CloudFront Functions

```bash
# Test URL rewriting
aws cloudfront test-function \
  --name hugo-url-rewrite \
  --if-match $(aws cloudfront describe-function --name hugo-url-rewrite --query 'ETag' --output text) \
  --event-object "$(cat <<'EOF'
{
  "version": "1.0",
  "context": {"eventType": "viewer-request"},
  "viewer": {"ip": "1.2.3.4"},
  "request": {
    "method": "GET",
    "uri": "/blog/my-post",
    "headers": {"host": {"value": "example.com"}},
    "querystring": {}
  }
}
EOF
)"
```

### Verify in Browser

```bash
# Check redirect behavior
curl -v https://example.com/blog/my-post 2>&1 | grep -i location
# Expected: Location: /blog/my-post/

# Check final URL resolves
curl -s -o /dev/null -w "%{http_code}" https://example.com/blog/my-post/
# Expected: 200

# Check file with extension (should NOT rewrite)
curl -s -o /dev/null -w "%{http_code}" https://example.com/css/main.css
# Expected: 200

# Check root
curl -s -o /dev/null -w "%{http_code}" https://example.com/
# Expected: 200
```

## Best Practices

**DO:**
- Use CloudFront Functions over Lambda@Edge for simple URL rewriting
- Redirect non-trailing-slash to trailing-slash for SEO consistency
- Handle both `/path` and `/path/` patterns
- Skip rewriting for files with extensions
- Test functions thoroughly before associating with distributions
- Use `cloudfront-js-2.0` runtime for CloudFront Functions

**DON'T:**
- Use Lambda@Edge for simple URL rewriting (overkill, more expensive)
- Forget to handle the root path (`/`)
- Rewrite URLs that have file extensions
- Create redirect loops (test thoroughly)
- Deploy Lambda@Edge outside us-east-1
- Skip the `publish = true` for Lambda@Edge versions

## Guidelines

### Essential

- CloudFront Function for `index.html` URL rewriting
- Handle trailing slash normalization
- Skip rewriting for static assets (files with extensions)
- Test with common URL patterns

### Recommended

- 301 redirect for trailing slash consistency
- WWW to non-www redirect at edge
- Custom redirect map for URL migrations
- Terraform for infrastructure management

### Advanced

- Lambda@Edge for geolocation routing
- A/B testing at the edge
- Dynamic security headers
- Origin request manipulation
- Multi-language routing

## Benefits

Clean URLs. Hugo pretty URLs work correctly on S3.

SEO. Consistent URL patterns with proper redirects.

Performance. Edge functions run in milliseconds at CloudFront locations.

Cost-Effective. CloudFront Functions cost $0.10/million invocations.

Flexibility. Handle complex routing without a server.

## Related

- [hugo-s3-deployment.md](./hugo-s3-deployment.md) - S3 bucket setup
- [hugo-cloudfront-integration.md](./hugo-cloudfront-integration.md) - CloudFront configuration
- [hugo-cicd-aws.md](./hugo-cicd-aws.md) - CI/CD pipeline
- [hugo-prerendering-strategies.md](./hugo-prerendering-strategies.md) - Build-time vs runtime rendering
