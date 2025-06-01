# Cache-Control Headers

HTTP header directing browsers and CDNs on how to cache responses. The single most impactful performance header.

## Why It Matters

- Correct caching eliminates redundant network requests (faster loads, lower bandwidth)
- Wrong caching serves stale content or wastes bandwidth re-fetching unchanged resources
- Different asset types need different caching strategies

## Recommended Values by Asset Type

| Asset Type | Cache-Control | Why |
|-----------|---------------|-----|
| HTML pages | `public, max-age=0, must-revalidate` | Always check for updates |
| CSS/JS (hashed filenames) | `public, max-age=31536000, immutable` | Content-addressed, never changes |
| Images (hashed) | `public, max-age=31536000, immutable` | Same as CSS/JS |
| Fonts | `public, max-age=31536000, immutable` | Rarely change |
| API responses | `private, no-cache` | User-specific, always fresh |
| robots.txt, sitemap | `public, max-age=86400` | Daily refresh is fine |
| .well-known files | `public, max-age=3600` | Hourly refresh |

## Key Directives

| Directive | Meaning |
|-----------|---------|
| `public` | CDNs and browsers can cache |
| `private` | Only the browser can cache (not CDNs) |
| `max-age=N` | Cache for N seconds |
| `s-maxage=N` | CDN cache time (overrides `max-age` for shared caches) |
| `no-cache` | Cache but always revalidate before use |
| `no-store` | Never cache (sensitive data) |
| `must-revalidate` | Once stale, must revalidate (no serving stale content) |
| `immutable` | Content will never change (skip revalidation) |
| `stale-while-revalidate=N` | Serve stale while fetching fresh in background |

## Static Site (Hugo + CloudFront) Pattern

```hcl
# HTML: always revalidate
ordered_cache_behavior {
  path_pattern = "*.html"
  default_ttl  = 0
}

# Hashed assets: cache forever
ordered_cache_behavior {
  path_pattern = "/assets/*"
  default_ttl  = 31536000
}
```

## NGINX

```nginx
# HTML
location ~* \.html$ {
    add_header Cache-Control "public, max-age=0, must-revalidate";
}

# Hashed static assets
location /assets/ {
    add_header Cache-Control "public, max-age=31536000, immutable";
}

# API
location /api/ {
    add_header Cache-Control "private, no-cache";
}
```

## S3 Upload with Cache Headers

```bash
# HTML
aws s3 sync public/ s3://bucket/ --exclude "*" --include "*.html" \
  --cache-control "public, max-age=0, must-revalidate"

# Assets
aws s3 sync public/assets/ s3://bucket/assets/ \
  --cache-control "public, max-age=31536000, immutable"
```

## Pitfalls to Avoid

- `max-age=31536000` on HTML pages (users see stale content for a year)
- `no-cache` on hashed assets (wastes bandwidth re-validating unchanged files)
- Confusing `no-cache` (revalidate before use) with `no-store` (never cache)
- Missing `immutable` on content-addressed assets (browsers still revalidate on reload)
- Forgetting `must-revalidate` on HTML (browsers may serve stale after `max-age` expires)
- `private` on public assets served through CDN (CDN will not cache them)

## Related

- [security-headers.md](./security-headers.md)
