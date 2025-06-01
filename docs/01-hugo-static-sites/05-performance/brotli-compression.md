# Brotli Compression for Hugo Static Sites

Enable Brotli as primary compression and Gzip as fallback for all text-based assets via CloudFront.

## Why It Matters

- Brotli produces 15-25% smaller output than Gzip for web content
- Reduces text-based transfer sizes by 70-85% (150KB page becomes ~32KB)
- Built-in 120KB web dictionary gives Brotli an inherent advantage for HTML/CSS/JS

## Brotli vs Gzip

| Metric | Gzip -9 | Brotli -11 | Savings |
|---|---|---|---|
| HTML 25KB | 7.5 KB | 6.2 KB | 17% smaller |
| CSS 45KB | 8.1 KB | 6.3 KB | 22% smaller |
| JS 80KB | 24 KB | 19.5 KB | 19% smaller |

Browser support: Brotli 97%+ (requires HTTPS). Gzip: universal.

## CloudFront Configuration

```hcl
# Cache policy for text-based content
parameters_in_cache_key_and_forwarded_to_origin {
  enable_accept_encoding_brotli = true
  enable_accept_encoding_gzip   = true
}

# Cache behavior
compress = true   # For HTML, CSS, JS, SVG, XML, JSON
compress = false  # For images, WOFF2 fonts (already compressed)
```

CloudFront stores separate Brotli, Gzip, and uncompressed cache entries per object.

## What to Compress

**Yes:** HTML, CSS, JS, JSON, XML, SVG, plain text, RSS/Atom feeds
**No:** JPEG, PNG, WebP, AVIF, WOFF2, WOFF, MP4, PDF, ZIP

## Minimum File Size

CloudFront does not compress files under 1,000 bytes. Inline sub-1KB assets directly in HTML instead.

```go-html-template
{{ if lt (len $css.Content) 1024 }}
  <style>{{ $css.Content | safeCSS }}</style>
{{ else }}
  <link rel="stylesheet" href="{{ $css.RelPermalink }}">
{{ end }}
```

## Hugo Asset Pipeline

```go-html-template
{{/* Hugo minifies at build -> CloudFront compresses at edge -> browser decompresses */}}
{{ $style := resources.Get "css/main.css" | postCSS }}
{{ if hugo.IsProduction }}
  {{ $style = $style | minify | fingerprint }}
{{ end }}
```

Minification and compression are complementary: minification removes syntactic redundancy, compression removes statistical redundancy.

## Verification

```bash
curl -sI -H "Accept-Encoding: br" https://example.com/ | grep content-encoding
# content-encoding: br

curl -sI -H "Accept-Encoding: gzip" https://example.com/ | grep content-encoding
# content-encoding: gzip
```

In DevTools, the Size column shows `6.2 kB / 25 kB` (transfer / actual). If they match, compression is not working.

## Build-Time Pre-Compression (Advanced)

CloudFront uses Brotli level ~4 at the edge. For maximum compression, pre-compress at level 11 during build (10-20% smaller). Trade-off: S3 cannot negotiate encoding, so most Hugo sites rely on CloudFront automatic compression for simplicity.

## Key Recommendations

- `compress = true` + both `enable_accept_encoding_*` for text content cache policies
- `compress = false` + both `enable_accept_encoding_*` disabled for images/fonts
- Minify with Hugo Pipes before CloudFront compresses
- `hugo --minify` for HTML output minification
- Inline assets under 1KB instead of compressing
- Keep Gzip enabled as fallback (~3% of traffic needs it)
- Verify `Content-Encoding: br` after every deployment

## Pitfalls

- Enabling compression for images/WOFF2 (already compressed, wastes CPU)
- Expecting CloudFront to compress files under 1KB (it does not)
- Disabling Gzip when enabling Brotli (breaks older clients)
- Skipping minification because "compression handles it" (they are complementary)
- Setting `compress = true` without also enabling encoding in cache policy
