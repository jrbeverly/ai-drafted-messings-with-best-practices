# Hugo Resource Fingerprinting

Content-based hashing for cache busting, Subresource Integrity (SRI), and long-term caching strategy.

## Why It Matters
- Fingerprinted filenames (`style.abc123.css`) let you set 1-year cache headers safely
- When file content changes, the hash changes, forcing browsers to fetch the new version
- SRI (`integrity` attribute) protects against tampered or compromised assets

## Basic Usage
```go-html-template
{{ $css := resources.Get "scss/main.scss" | toCSS | minify | fingerprint }}
<link rel="stylesheet" href="{{ $css.Permalink }}" integrity="{{ $css.Data.Integrity }}">
```
Output: `/css/main.a8f4e2d1b6c9.css` with `integrity="sha256-..."`

## Hash Algorithms
```go-html-template
{{ $css | fingerprint }}            <!-- SHA-256 (default) -->
{{ $css | fingerprint "sha384" }}   <!-- SHA-384 (recommended for SRI) -->
{{ $css | fingerprint "sha512" }}   <!-- SHA-512 -->
```

## Environment-Specific
```go-html-template
{{ if hugo.IsProduction }}
  {{ $css = $css | minify | fingerprint "sha384" }}
  <link rel="stylesheet" href="{{ $css.Permalink }}" integrity="{{ $css.Data.Integrity }}" crossorigin="anonymous">
{{ else }}
  <link rel="stylesheet" href="{{ $css.RelPermalink }}">
{{ end }}
```

## Caching Strategy
- **Fingerprinted assets:** `Cache-Control: public, max-age=31536000, immutable` (1 year)
- **HTML files:** `Cache-Control: public, max-age=0, must-revalidate` (always revalidate)
- Apply via CDN config (CloudFront), server config (NGINX), or hosting headers (Netlify)

## S3 Deploy Pattern
```bash
# Assets: long cache
aws s3 sync public/ s3://bucket --cache-control "public, max-age=31536000" --exclude "*.html"
# HTML: no cache
aws s3 sync public/ s3://bucket --cache-control "no-cache" --exclude "*" --include "*.html"
```

## Pitfalls
- Don't hard-code asset paths -- always use `.Permalink` or `.RelPermalink` from the fingerprinted resource
- Don't fingerprint HTML files (they need to be re-fetched to discover new asset hashes)
- Don't skip fingerprinting in production (stale cached assets cause subtle bugs)
- Don't forget `crossorigin="anonymous"` when using SRI with CDN-served assets
