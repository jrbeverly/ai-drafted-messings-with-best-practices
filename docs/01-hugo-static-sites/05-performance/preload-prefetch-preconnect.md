# Preload, Prefetch & Preconnect

Tell the browser about critical resources before it discovers them naturally during HTML parsing.

## Why It Matters

- Preload closes the discovery gap for fonts, CSS, and hero images (saves 200-500ms)
- Preconnect saves 150-300ms per third-party origin (DNS + TCP + TLS)
- Prefetch makes next-page navigation feel instant (loaded from cache)

## The Five Hint Types

| Hint | What It Does | When Needed | Cost |
|---|---|---|---|
| `preload` | Fetches resource at high priority | Current page (now) | High |
| `prefetch` | Fetches resource at low priority | Next page (idle time) | Low |
| `preconnect` | DNS + TCP + TLS handshake | Current page (third-party) | Medium |
| `dns-prefetch` | DNS resolution only | Current page (non-critical) | Minimal |
| Speculation Rules | Prerenders entire page | Next page (very likely) | Very high |

## Preload (2-3 resources max)

```html
<link rel="preload" href="/fonts/inter-regular.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/css/main.css" as="style">
<link rel="preload" href="/images/hero.webp" as="image" fetchpriority="high">
```

- `as` attribute is **required** (wrong/missing causes double download)
- `crossorigin` is **required** on fonts (even same-origin)
- `type="font/woff2"` lets unsupporting browsers skip the preload

## Preconnect + DNS-Prefetch

```html
<link rel="preconnect" href="https://api.example.com" crossorigin>
<link rel="dns-prefetch" href="https://api.example.com">  <!-- fallback -->
```

Limit to 2-4 origins. Each preconnect consumes a connection slot.

## Prefetch

```go-html-template
{{ if .Paginator }}{{ if .Paginator.HasNext }}
  <link rel="prefetch" href="{{ .Paginator.Next.URL }}">
{{ end }}{{ end }}
{{ with .NextInSection }}
  <link rel="prefetch" href="{{ .RelPermalink }}">
{{ end }}
```

## Speculation Rules API (Chromium only)

```html
<script type="speculationrules">
{ "prerender": [{ "where": { "href_matches": "/blog/*" }, "eagerness": "moderate" }],
  "prefetch":  [{ "where": { "href_matches": "/*" }, "eagerness": "conservative" }] }
</script>
```

## Hugo Fingerprinted Assets

Process once, reference same variable for both preload and usage:

```go-html-template
{{ $css := resources.Get "css/main.css" | postCSS | minify | fingerprint }}
<link rel="preload" href="{{ $css.RelPermalink }}" as="style">
<link rel="stylesheet" href="{{ $css.RelPermalink }}" integrity="{{ $css.Data.Integrity }}">
```

## Ordering in `<head>`

1. `preconnect` + `dns-prefetch` (establish connections)
2. `preload` (fetch critical resources)
3. Stylesheets (use preloaded resources)
4. `prefetch` (low priority, idle time)
5. Scripts (`defer`)

## Key Recommendations

- Preload only 2-3 truly critical resources per page
- Always include `as` and `crossorigin` (for fonts) on preloads
- Pair every `preconnect` with a `dns-prefetch` fallback
- Make preloads conditional based on page type
- Use Hugo variables (not hardcoded URLs) for fingerprinted assets

## Pitfalls

- Preloading more than 3-4 resources (dilutes priority)
- Using `preload` for next-page resources (use `prefetch`)
- Using `prefetch` for current-page resources (use `preload`)
- Omitting `crossorigin` on font preloads (double download)
- Preloading resources not used on current page (wasted bandwidth + console warning)
