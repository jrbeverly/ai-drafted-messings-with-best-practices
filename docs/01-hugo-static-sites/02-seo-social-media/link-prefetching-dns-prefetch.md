# Link Prefetching and DNS Prefetch

Resource hints. Preconnect. DNS prefetch. Prefetch. Preload. Performance optimization.

## Principle

Use resource hints to improve page load performance. Establish early connections, resolve DNS, and prefetch resources before they're needed. Reduce perceived latency for users navigating your site.

## Resource Hint Types

### Overview

| Hint | Purpose | When to Use |
|------|---------|-------------|
| dns-prefetch | Resolve DNS early | Third-party domains you'll use |
| preconnect | DNS + TCP + TLS | Critical third-party origins |
| prefetch | Fetch resource for future use | Resources for next page |
| preload | Fetch resource for current page | Critical resources (fonts, CSS) |
| prerender | Pre-render entire page | Highly likely next page |

### Priority Order

```
preload → Current page, critical (highest priority)
preconnect → Current page, third-party origin (high priority)
dns-prefetch → Possible future use (medium priority)
prefetch → Future navigation (low priority)
prerender → Future navigation, full page (lowest priority)
```

## DNS Prefetch

**Resolve DNS for domains before they're needed.**

### Syntax

```html
<link rel="dns-prefetch" href="https://analytics.example.com">
<link rel="dns-prefetch" href="https://fonts.googleapis.com">
<link rel="dns-prefetch" href="https://cdn.example.com">
```

### When to Use

**DNS prefetch for:**
- Analytics services
- Third-party widgets
- CDN domains (if not critical)
- Social media embeds
- Ad networks
- Any domain that might be requested

**DNS resolution takes:** 20-120ms (varies by network)

### Hugo Implementation

```go-html-template
{{ range .Site.Params.dns_prefetch }}
  <link rel="dns-prefetch" href="{{ . }}">
{{ end }}
```

**config.toml:**

```toml
[params]
  dns_prefetch = [
    "https://analytics.example.com",
    "https://fonts.googleapis.com",
    "https://www.google-analytics.com"
  ]
```

## Preconnect

**Establish full connection (DNS + TCP + TLS) to an origin.**

### Syntax

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="preconnect" href="https://cdn.example.com" crossorigin>
```

### When to Use

**Preconnect for:**
- Font origins (Google Fonts, Adobe Fonts)
- CDN origins
- API endpoints
- Any critical third-party origin

**Connection savings:** 100-500ms (DNS + TCP + TLS)

**Important:** Limit to 2-4 origins. Too many preconnects waste resources.

### crossorigin Attribute

```html
<!-- Same-origin or no CORS -->
<link rel="preconnect" href="https://example.com">

<!-- Cross-origin with CORS (fonts, APIs) -->
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

<!-- With credentials -->
<link rel="preconnect" href="https://api.example.com" crossorigin="use-credentials">
```

**Use crossorigin for:**
- Font files (always CORS)
- CORS-enabled API endpoints
- Resources served with `Access-Control-Allow-Origin`

### Hugo Implementation

```go-html-template
{{ range .Site.Params.preconnect }}
  <link rel="preconnect" href="{{ . }}" crossorigin>
{{ end }}
```

**config.toml:**

```toml
[params]
  preconnect = [
    "https://fonts.googleapis.com",
    "https://fonts.gstatic.com"
  ]
```

## Prefetch

**Download resources that will be needed for future navigation.**

### Syntax

```html
<!-- Prefetch next page -->
<link rel="prefetch" href="/blog/page/2/">

<!-- Prefetch CSS for next page -->
<link rel="prefetch" href="/css/blog.css" as="style">

<!-- Prefetch JavaScript -->
<link rel="prefetch" href="/js/comments.js" as="script">

<!-- Prefetch image -->
<link rel="prefetch" href="/images/hero-next.jpg" as="image">
```

### When to Use

**Prefetch for:**
- Likely next page (pagination)
- Resources for common user paths
- Non-critical CSS/JS for interactions
- Images that will be needed soon

**Timing:** Browser downloads during idle time (low priority)

### Hugo Implementation

**Prefetch next/previous pages:**

```go-html-template
{{ if .Paginator }}
  {{ if .Paginator.HasNext }}
    <link rel="prefetch" href="{{ .Paginator.Next.URL | absURL }}">
  {{ end }}
{{ end }}

{{/* Prefetch related articles */}}
{{ $related := .Site.RegularPages.Related . | first 3 }}
{{ range $related }}
  <link rel="prefetch" href="{{ .Permalink }}">
{{ end }}
```

### Dynamic Prefetching (JavaScript)

**Prefetch on hover (aggressive strategy):**

```html
<script>
document.addEventListener('DOMContentLoaded', () => {
  // Prefetch links on hover
  document.querySelectorAll('a[href^="/"]').forEach(link => {
    link.addEventListener('mouseenter', () => {
      const prefetch = document.createElement('link');
      prefetch.rel = 'prefetch';
      prefetch.href = link.href;
      document.head.appendChild(prefetch);
    }, { once: true });
  });
});
</script>
```

## Preload

**Download resources needed for the current page with high priority.**

### Syntax

```html
<!-- Preload font -->
<link rel="preload" href="/fonts/inter-var.woff2" as="font" type="font/woff2" crossorigin>

<!-- Preload CSS -->
<link rel="preload" href="/css/critical.css" as="style">

<!-- Preload hero image -->
<link rel="preload" href="/images/hero.webp" as="image" type="image/webp">

<!-- Preload JavaScript -->
<link rel="preload" href="/js/app.js" as="script">
```

### The "as" Attribute

**Required for preload. Tells browser the resource type:**

| as Value | Resource Type |
|----------|---------------|
| font | Font files (.woff2) |
| style | CSS stylesheets |
| script | JavaScript files |
| image | Images |
| fetch | Resources fetched with fetch() |
| document | HTML documents (iframes) |
| audio | Audio files |
| video | Video files |

### When to Use

**Preload for:**
- Web fonts (critical for rendering)
- Above-the-fold images (hero images)
- Critical CSS
- JavaScript needed for initial render
- LCP (Largest Contentful Paint) image

**Do NOT preload:**
- Below-the-fold images
- Non-critical CSS/JS
- Resources that may not be used
- Too many resources (3-5 max)

### Hugo Implementation

**Preload fonts:**

```go-html-template
{{/* Preload critical fonts */}}
{{ range .Site.Params.preload_fonts }}
  <link rel="preload" href="{{ . }}" as="font" type="font/woff2" crossorigin>
{{ end }}
```

**Preload hero image:**

```go-html-template
{{/* Preload hero image for current page */}}
{{ with .Params.hero_image }}
  {{ $image := resources.Get . }}
  {{ with $image }}
    {{ $hero := .Fill "1200x600 center webp q85" }}
    <link rel="preload" href="{{ $hero.Permalink }}" as="image" type="image/webp">
  {{ end }}
{{ end }}
```

**Preload critical CSS:**

```go-html-template
{{ $critical := resources.Get "css/critical.css" | minify | fingerprint }}
<link rel="preload" href="{{ $critical.Permalink }}" as="style">
<link rel="stylesheet" href="{{ $critical.Permalink }}">
```

## Prerender

**Pre-render an entire page in background.**

### Syntax

```html
<link rel="prerender" href="https://example.com/next-page/">
```

### Speculation Rules (Modern Alternative)

**Chrome 109+ supports Speculation Rules API:**

```html
<script type="speculationrules">
{
  "prerender": [
    {
      "where": {
        "href_matches": "/blog/*"
      },
      "eagerness": "moderate"
    }
  ],
  "prefetch": [
    {
      "where": {
        "href_matches": "/*"
      },
      "eagerness": "conservative"
    }
  ]
}
</script>
```

**Eagerness levels:**
- `conservative` - On click (safest)
- `moderate` - On hover
- `eager` - Immediately (most aggressive)

## Complete Hugo Template

**layouts/partials/head/resource-hints.html:**

```go-html-template
{{/* === Preconnect (critical third-party origins) === */}}
{{ range .Site.Params.preconnect }}
  <link rel="preconnect" href="{{ . }}" crossorigin>
{{ end }}

{{/* === DNS Prefetch (non-critical origins) === */}}
{{ range .Site.Params.dns_prefetch }}
  <link rel="dns-prefetch" href="{{ . }}">
{{ end }}

{{/* === Preload (critical current-page resources) === */}}
{{/* Fonts */}}
{{ range .Site.Params.preload_fonts }}
  <link rel="preload" href="{{ . }}" as="font" type="font/woff2" crossorigin>
{{ end }}

{{/* Hero image */}}
{{ with .Params.hero_image }}
  {{ $image := resources.Get . }}
  {{ with $image }}
    {{ $hero := .Fill "1200x600 center webp q85" }}
    <link rel="preload" href="{{ $hero.Permalink }}" as="image" type="image/webp">
  {{ end }}
{{ end }}

{{/* === Prefetch (likely next navigation) === */}}
{{ if .Paginator }}
  {{ if .Paginator.HasNext }}
    <link rel="prefetch" href="{{ .Paginator.Next.URL | absURL }}">
  {{ end }}
{{ end }}
```

**config.toml:**

```toml
[params]
  # Critical origins (2-4 max)
  preconnect = [
    "https://fonts.googleapis.com",
    "https://fonts.gstatic.com"
  ]

  # Non-critical origins
  dns_prefetch = [
    "https://www.google-analytics.com",
    "https://cdn.example.com"
  ]

  # Critical fonts to preload
  preload_fonts = [
    "/fonts/inter-var-latin.woff2"
  ]
```

## Best Practices

### General

**✅ DO:**
- Use preconnect for critical third-party origins (2-4 max)
- Use dns-prefetch for non-critical origins
- Preload critical fonts and LCP image
- Prefetch likely next pages during idle time
- Place resource hints early in `<head>`

**❌ DON'T:**
- Preconnect to more than 4-5 origins
- Preload non-critical resources
- Prefetch resources that won't be used
- Forget the `as` attribute on preload
- Forget `crossorigin` on font preloads
- Use prerender for uncertain pages

### Performance Budget

**Recommended limits:**
- Preconnect: 2-4 origins
- DNS prefetch: 4-8 origins
- Preload: 3-5 resources
- Prefetch: 2-5 pages/resources

### Order in `<head>`

```html
<head>
  <!-- 1. Meta charset (first) -->
  <meta charset="utf-8">

  <!-- 2. Preconnect (establish connections early) -->
  <link rel="preconnect" href="https://fonts.googleapis.com">

  <!-- 3. DNS prefetch -->
  <link rel="dns-prefetch" href="https://analytics.example.com">

  <!-- 4. Preload (critical resources) -->
  <link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin>

  <!-- 5. Stylesheets -->
  <link rel="stylesheet" href="/css/main.css">

  <!-- 6. Prefetch (future navigation, lowest priority) -->
  <link rel="prefetch" href="/next-page/">
</head>
```

## Testing

### Chrome DevTools

**Network panel:**
1. Open DevTools (F12)
2. Go to Network tab
3. Look for "Priority" column
4. Verify preloaded resources load first
5. Check connection timing for preconnected origins

**Performance panel:**
1. Record page load
2. Check Connection Start timing
3. Verify preconnect saves connection time

### Lighthouse

**Run Lighthouse audit:**
- Check "Preconnect to required origins"
- Check "Preload Largest Contentful Paint image"
- Review resource loading waterfall

### Web Vitals

**Monitor impact on:**
- LCP (Largest Contentful Paint) - preload helps
- FCP (First Contentful Paint) - preconnect helps
- TTFB (Time to First Byte) - dns-prefetch helps

## Guidelines

### Essential

**Minimum for performance:**
- Preconnect to font origins (if using web fonts)
- DNS prefetch for analytics
- Preload critical fonts

### Recommended

**For better performance:**
- Preload LCP image
- Prefetch next pagination pages
- dns-prefetch for all third-party origins
- Preconnect to CDN
- Place hints early in `<head>`

### Advanced

**For maximum performance:**
- Dynamic prefetching on hover
- Speculation Rules API
- Critical CSS preloading
- Module preloading
- Performance monitoring

## Benefits

Faster Loading. Reduce connection and download latency.

Better UX. Instant page transitions with prefetch.

Core Web Vitals. Improve LCP, FCP, and perceived performance.

Zero Cost. Browser handles hints efficiently, no wasted resources.

Progressive. Hints are suggestions, browsers can ignore if under load.

## Related

- [rel-attributes.md](./rel-attributes.md) - All rel attribute types
- [meta-tags-comprehensive.md](./meta-tags-comprehensive.md) - Complete head template
- [../../01-hugo-basics/hugo-asset-pipeline.md](../01-hugo-basics/hugo-asset-pipeline.md) - Hugo Pipes for assets
