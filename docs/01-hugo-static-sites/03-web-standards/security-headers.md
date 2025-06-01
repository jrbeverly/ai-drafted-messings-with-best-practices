# Security Headers

HTTP response headers that instruct browsers to apply security policies. Defense-in-depth against XSS, clickjacking, MIME sniffing, and protocol downgrade attacks.

## Why It Matters

- Browsers enforce these policies automatically once headers are set
- A+ grade on SecurityHeaders.com requires all six essential headers
- Set once at the CDN/server level, protects every page

## Essential Header Set

```
Content-Security-Policy: default-src 'self'; frame-ancestors 'none'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
```

## Header Quick Reference

| Header | Value | Prevents |
|--------|-------|----------|
| `Content-Security-Policy` | `default-src 'self'` | XSS, data injection |
| `X-Frame-Options` | `DENY` | Clickjacking |
| `X-Content-Type-Options` | `nosniff` | MIME type sniffing |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | SSL stripping, HTTP downgrade |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Referrer information leakage |
| `Permissions-Policy` | `geolocation=(), camera=()` | Unauthorized feature access |

## Referrer-Policy Values

| Value | Behavior |
|-------|----------|
| `no-referrer` | Never send referrer |
| `strict-origin-when-cross-origin` | Full URL same-origin, origin-only cross-origin (recommended) |
| `same-origin` | Only send to same origin |
| `origin` | Always send origin only |

## NGINX

```nginx
add_header Content-Security-Policy "default-src 'self'; frame-ancestors 'none'" always;
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

Use `always` to send headers on error pages (4xx, 5xx) too.

## CloudFront (Terraform)

```hcl
resource "aws_cloudfront_response_headers_policy" "security" {
  name = "security-headers"
  security_headers_config {
    content_security_policy {
      override                = true
      content_security_policy = "default-src 'self'; frame-ancestors 'none'"
    }
    frame_options       { override = true; frame_option = "DENY" }
    content_type_options { override = true }
    strict_transport_security {
      override                   = true
      access_control_max_age_sec = 31536000
      include_subdomains         = true
      preload                    = true
    }
    referrer_policy {
      override        = true
      referrer_policy = "strict-origin-when-cross-origin"
    }
  }
  custom_headers_config {
    items {
      header = "Permissions-Policy"
      value  = "geolocation=(), microphone=(), camera=()"
      override = true
    }
  }
}
```

## HSTS Preload

To submit to the browser preload list (HTTPS enforced before first visit):

1. Serve valid HTTPS certificate
2. Redirect all HTTP to HTTPS
3. Include `max-age >= 31536000`, `includeSubDomains`, and `preload`
4. Submit at https://hstspreload.org/

**Warning:** Removal from the preload list is slow. Commit for 1+ year.

## Testing

- **SecurityHeaders.com:** https://securityheaders.com/ (letter grade)
- **Mozilla Observatory:** https://observatory.mozilla.org/
- **CLI:** `curl -I https://example.com | grep -i "content-security\|x-frame\|strict-transport\|referrer-policy"`

## Pitfalls to Avoid

- HSTS without actually having HTTPS configured (locks users out)
- Overly strict CSP breaking site functionality (test with report-only first)
- Headers only on homepage (use `always` in NGINX, apply at CDN level)
- Forgetting error pages (4xx/5xx still need security headers)
- Using deprecated `X-XSS-Protection` as sole XSS defense (use CSP instead)

## Related

- [csp-content-security-policy.md](./csp-content-security-policy.md)
- [permissions-policy.md](./permissions-policy.md)
