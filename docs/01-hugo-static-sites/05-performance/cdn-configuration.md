# CDN Configuration for Hugo Static Sites

Configure CloudFront with content-type-aware caching, compression negotiation, and security headers for S3-hosted Hugo sites.

## Why It Matters

- Sub-100ms global TTFB via edge caching from 400+ locations
- Fingerprinted assets never need invalidation (new hash = new URL)
- Automatic Brotli/Gzip negotiation reduces transfer sizes 70-85%

## Two-Tier Caching Model

| Content Type | Cache-Control | Rationale |
|---|---|---|
| HTML, sitemap, RSS | `public, max-age=300, s-maxage=3600` | Changes on deploy, short TTL |
| Fingerprinted CSS/JS/fonts | `public, max-age=31536000, immutable` | Hash in filename, never changes |
| Images | `public, max-age=86400, s-maxage=604800` | Rarely change |

## Hugo Deploy Matchers

```toml
# hugo.toml
[[deployment.matchers]]
  pattern = "^.+\\.html$"
  cacheControl = "public, max-age=300, s-maxage=3600"
  gzip = true
[[deployment.matchers]]
  pattern = "^.+\\.(css|js)$"
  cacheControl = "public, max-age=31536000, immutable"
  gzip = true
[[deployment.matchers]]
  pattern = "^.+\\.(woff2|woff)$"
  cacheControl = "public, max-age=31536000, immutable"
```

## Compression Rules

- `compress = true` for HTML, CSS, JS, SVG, XML, JSON
- `compress = false` for images (already compressed) and WOFF2 (Brotli internally)
- Enable both `enable_accept_encoding_brotli` and `enable_accept_encoding_gzip` in cache policies

## Cache Invalidation

Only invalidate mutable content after deploy. Fingerprinted assets use new URLs automatically.

```bash
aws cloudfront create-invalidation --distribution-id "$ID" \
  --paths "/index.html" "/*/index.html" "/sitemap.xml" "/robots.txt"
```

## CloudFront Key Settings

```hcl
http_version        = "http2and3"
price_class         = "PriceClass_100"  # Start here, upgrade with traffic data
viewer_protocol_policy = "redirect-to-https"
minimum_protocol_version = "TLSv1.2_2021"
```

## Error Pages

Map S3 403 to 404 page (OAC returns 403 for missing objects):

```hcl
custom_error_response {
  error_code = 403; response_code = 404
  response_page_path = "/404.html"; error_caching_min_ttl = 300
}
```

## Security Headers

Apply via `aws_cloudfront_response_headers_policy`: HSTS, X-Frame-Options: DENY, X-Content-Type-Options: nosniff, Referrer-Policy, CSP, Permissions-Policy.

## Hugo baseURL

Set to CDN custom domain, never S3 bucket URL. Use `relURL` for internal links, `absURL` only for sitemaps/RSS/OG tags.

## Key Recommendations

- Separate cache policies for mutable (HTML) and immutable (fingerprinted) content
- Use CloudFront Functions for WWW redirect and trailing slash normalization
- Start with `PriceClass_100`, upgrade based on geographic traffic
- Attach security headers policy to all cache behaviors
- Monitor cache hit rate (target >90%) and 5xx error rate (<0.1%)

## Pitfalls

- Wildcard invalidation (`/*`) on every deploy (unnecessary for fingerprinted assets)
- Long TTLs on HTML files (stale content for hours)
- Compressing images or WOFF2 (already compressed)
- Setting `baseURL` to S3 bucket or `d1234.cloudfront.net`
- Skipping 403-to-404 error mapping
