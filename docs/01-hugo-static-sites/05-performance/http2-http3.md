# HTTP/2 and HTTP/3 for Hugo Static Sites

Serve Hugo sites over HTTP/2 and HTTP/3 for multiplexed resource loading, zero head-of-line blocking, and 0-RTT connection resumption.

## Why It Matters

- HTTP/2 multiplexing loads all resources in parallel over one connection (vs 6 serial connections in HTTP/1.1)
- HTTP/3 (QUIC) eliminates transport-level head-of-line blocking on lossy/mobile networks
- 0-RTT resumption allows returning visitors to send requests with zero round-trip delay

## Enable on CloudFront

```hcl
resource "aws_cloudfront_distribution" "hugo_site" {
  http_version = "http2and3"  # Negotiates highest supported protocol
  viewer_certificate {
    minimum_protocol_version = "TLSv1.2_2021"  # Required for ALPN + TLS 1.3
  }
}
```

CloudFront sends `Alt-Svc: h3=":443"; ma=86400` to advertise HTTP/3. Clients fall back to HTTP/2 if QUIC is blocked.

## Domain Sharding Is Obsolete

Serve all assets from a single origin. Domain sharding hurts HTTP/2 performance by preventing multiplexing and adding DNS/TLS overhead per domain.

## HTTP/2-Aware Bundling Strategy

Bundle by **change frequency and usage scope**, not to minimize request count:

```go-html-template
{{ $globalCSS := resources.Get "css/global.css" | postCSS | minify | fingerprint }}
{{ if eq .Section "blog" }}
  {{ $blogCSS := resources.Get "css/blog.css" | postCSS | minify | fingerprint }}
{{ end }}
```

Target: 2-5 CSS files and 1-3 JS files per page. Balances compression efficiency with cache granularity.

## Priority Hints (fetchpriority)

```html
<img src="hero.webp" fetchpriority="high">   <!-- LCP image -->
<img src="thumb.webp" fetchpriority="low" loading="lazy">  <!-- Below fold -->
<script src="analytics.js" defer fetchpriority="low"></script>
```

Use `high` on exactly one resource (LCP). Overusing it cancels the benefit.

## Server Push Is Deprecated

Chrome removed HTTP/2 server push in v106. Use `<link rel="preload">` in `<head>` instead.

## Testing

```bash
curl -sI --http2 https://example.com/ | head -1        # HTTP/2 200
curl -sI https://example.com/ | grep -i alt-svc         # h3=":443"
curl -sI --http3 https://example.com/ | head -1         # HTTP/3 200
```

Hugo's dev server only supports HTTP/1.1. Test protocols on staging CloudFront.

## Key Recommendations

- `http_version = "http2and3"` on all CloudFront distributions
- Single origin for all assets (no domain sharding)
- `fetchpriority="high"` on the LCP image only
- `<link rel="preload">` for critical CSS and fonts
- Wildcard TLS certificate for connection coalescing across subdomains
- 0-RTT is safe for static sites (GET requests are idempotent)

## Pitfalls

- Domain sharding across subdomains (counterproductive with HTTP/2)
- Splitting into dozens of micro-files (compression ratio suffers)
- Bundling everything into one monolithic file (cache invalidation suffers)
- Using `fetchpriority="high"` on every resource
- Testing protocol performance on Hugo dev server (HTTP/1.1 only)
