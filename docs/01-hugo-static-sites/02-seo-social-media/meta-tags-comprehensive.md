# Comprehensive Meta Tags

Complete meta tag reference. HTML head elements. Viewport. Robots. Theme color. Author. All meta tags for Hugo sites.

## Principle

Include all necessary meta tags in your HTML head for proper rendering, SEO, browser behavior, and social sharing. Use Hugo templates for consistent generation across all pages. Organize meta tags by category for maintainability.

## Essential Meta Tags

### Character Encoding

```html
<meta charset="utf-8">
```

**Purpose:** Specify character encoding (always UTF-8)
**Required:** Yes, must be first element in `<head>`

### Viewport

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

**Purpose:** Responsive design, mobile rendering
**Required:** Yes, for all responsive sites

**Options:**

```html
<!-- Standard responsive -->
<meta name="viewport" content="width=device-width, initial-scale=1">

<!-- Prevent user zoom (not recommended) -->
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1">

<!-- Allow user zoom (recommended) -->
<meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
```

### Title

```html
<title>Page Title | Brand Name</title>
```

**Purpose:** Page title in browser tab and search results
**Required:** Yes
**Limits:** 50-60 characters

### Description

```html
<meta name="description" content="Page description for search results and social sharing.">
```

**Purpose:** Search result snippet, social sharing description
**Recommended:** Yes (120-160 characters)

## SEO Meta Tags

### Robots

```html
<meta name="robots" content="index, follow">
```

**Purpose:** Control search engine crawling and indexing

**Values:**

| Directive | Meaning |
|-----------|---------|
| index | Allow indexing |
| noindex | Don't index this page |
| follow | Follow links on this page |
| nofollow | Don't follow links |
| noarchive | Don't show cached version |
| nosnippet | Don't show snippet in results |
| noimageindex | Don't index images |
| max-snippet:N | Max snippet length (characters) |
| max-image-preview:large | Allow large image preview |
| max-video-preview:N | Max video preview length (seconds) |

**Common combinations:**

```html
<!-- Default (index and follow) -->
<meta name="robots" content="index, follow">

<!-- Don't index but follow links -->
<meta name="robots" content="noindex, follow">

<!-- Index but don't follow links -->
<meta name="robots" content="index, nofollow">

<!-- Don't index or follow -->
<meta name="robots" content="noindex, nofollow">

<!-- Allow large previews -->
<meta name="robots" content="index, follow, max-image-preview:large, max-snippet:-1, max-video-preview:-1">
```

**Target specific bots:**

```html
<!-- Google only -->
<meta name="googlebot" content="noindex">

<!-- Bing only -->
<meta name="bingbot" content="noindex">
```

### Author

```html
<meta name="author" content="Jane Doe">
```

**Purpose:** Page author attribution

### Keywords (Deprecated)

```html
<!-- ❌ NOT RECOMMENDED - Google ignores this -->
<meta name="keywords" content="hugo, static site, performance">
```

**Note:** Google has not used meta keywords for ranking since 2009. Bing may give minor weight.

### Canonical

```html
<link rel="canonical" href="https://example.com/page/">
```

**Purpose:** Preferred URL for this page
**Required:** Yes (see [canonical-urls.md](./canonical-urls.md))

### Referrer Policy

```html
<meta name="referrer" content="strict-origin-when-cross-origin">
```

**Purpose:** Control referrer information sent with requests

**Values:**

| Value | Behavior |
|-------|----------|
| no-referrer | Never send referrer |
| no-referrer-when-downgrade | Send on HTTPS→HTTPS, not HTTPS→HTTP |
| origin | Send origin only (no path) |
| origin-when-cross-origin | Full URL same-origin, origin only cross-origin |
| strict-origin-when-cross-origin | Recommended default |
| same-origin | Full URL same-origin, nothing cross-origin |
| unsafe-url | Always send full URL |

## Browser and Display

### Theme Color

```html
<meta name="theme-color" content="#1a1a2e">
```

**Purpose:** Browser chrome color (mobile browsers, PWA)

**With media queries (dark/light mode):**

```html
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#1a1a2e" media="(prefers-color-scheme: dark)">
```

### Color Scheme

```html
<meta name="color-scheme" content="light dark">
```

**Purpose:** Tell browser which color schemes page supports

**Values:**

```html
<meta name="color-scheme" content="light">       <!-- Light only -->
<meta name="color-scheme" content="dark">         <!-- Dark only -->
<meta name="color-scheme" content="light dark">   <!-- Both, prefers light -->
<meta name="color-scheme" content="dark light">   <!-- Both, prefers dark -->
```

### Format Detection

```html
<meta name="format-detection" content="telephone=no">
```

**Purpose:** Prevent auto-detection of phone numbers, emails, addresses

```html
<!-- Disable all auto-detection (iOS Safari) -->
<meta name="format-detection" content="telephone=no, date=no, email=no, address=no">
```

### IE Compatibility (Legacy)

```html
<!-- Force latest IE rendering engine -->
<meta http-equiv="X-UA-Compatible" content="IE=edge">
```

**Note:** Only needed if supporting IE11. Can be omitted for modern sites.

## Social Media Meta Tags

### Open Graph

```html
<meta property="og:type" content="article">
<meta property="og:url" content="https://example.com/page/">
<meta property="og:title" content="Page Title">
<meta property="og:description" content="Page description for social sharing.">
<meta property="og:image" content="https://example.com/images/og-image.jpg">
<meta property="og:site_name" content="Site Name">
<meta property="og:locale" content="en_US">
```

**See:** [open-graph-meta-tags.md](./open-graph-meta-tags.md)

### Twitter Card

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@username">
<meta name="twitter:title" content="Page Title">
<meta name="twitter:description" content="Page description.">
<meta name="twitter:image" content="https://example.com/images/twitter-card.jpg">
```

**See:** [twitter-card-types.md](./twitter-card-types.md)

## Link Elements

### Favicon

```html
<link rel="icon" href="/favicon.ico" sizes="32x32">
<link rel="icon" href="/icon.svg" type="image/svg+xml">
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
```

### Preconnect

```html
<!-- Preconnect to external origins for faster loading -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="preconnect" href="https://cdn.example.com">
```

### DNS Prefetch

```html
<link rel="dns-prefetch" href="https://fonts.googleapis.com">
<link rel="dns-prefetch" href="https://analytics.example.com">
```

### Preload

```html
<!-- Preload critical resources -->
<link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/css/critical.css" as="style">
```

### Stylesheet

```html
<link rel="stylesheet" href="/css/main.css">
```

### RSS/Atom Feed

```html
<link rel="alternate" type="application/rss+xml" title="RSS Feed" href="/index.xml">
<link rel="alternate" type="application/atom+xml" title="Atom Feed" href="/atom.xml">
```

### Web Manifest

```html
<link rel="manifest" href="/site.webmanifest">
```

### Hreflang

```html
<link rel="alternate" hreflang="en" href="https://example.com/en/page/">
<link rel="alternate" hreflang="es" href="https://example.com/es/page/">
<link rel="alternate" hreflang="x-default" href="https://example.com/en/page/">
```

**See:** [hreflang-tags.md](./hreflang-tags.md)

### Sitemap

```html
<!-- Not standard, but some crawlers look for it -->
<link rel="sitemap" type="application/xml" href="/sitemap.xml">
```

## Security Meta Tags

### Content Security Policy

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:">
```

**Note:** Prefer HTTP headers over meta tags for CSP.

### X-Content-Type-Options

**HTTP header (not meta tag):**

```
X-Content-Type-Options: nosniff
```

### Permissions Policy

**HTTP header:**

```
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

## Verification Tags

### Google Search Console

```html
<meta name="google-site-verification" content="your-verification-code">
```

### Bing Webmaster Tools

```html
<meta name="msvalidate.01" content="your-verification-code">
```

### Pinterest

```html
<meta name="p:domain_verify" content="your-verification-code">
```

### Yandex

```html
<meta name="yandex-verification" content="your-verification-code">
```

## Hugo Complete Template

### Full Head Template

**layouts/partials/head/meta-tags.html:**

```go-html-template
{{/* Essential */}}
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">

{{/* Title */}}
{{ partial "head/title.html" . }}

{{/* Description */}}
{{ partial "head/description.html" . }}

{{/* Author */}}
{{ with .Params.author }}
  <meta name="author" content="{{ . }}">
{{ else }}
  {{ with .Site.Params.author }}
    <meta name="author" content="{{ . }}">
  {{ end }}
{{ end }}

{{/* Robots */}}
{{ if .Params.noindex }}
  <meta name="robots" content="noindex, follow">
{{ else }}
  <meta name="robots" content="index, follow, max-image-preview:large, max-snippet:-1, max-video-preview:-1">
{{ end }}

{{/* Referrer */}}
<meta name="referrer" content="strict-origin-when-cross-origin">

{{/* Theme color */}}
{{ with .Site.Params.theme_color }}
  <meta name="theme-color" content="{{ . }}">
{{ end }}
{{ with .Site.Params.theme_color_dark }}
  <meta name="theme-color" content="{{ . }}" media="(prefers-color-scheme: dark)">
{{ end }}

{{/* Color scheme */}}
<meta name="color-scheme" content="light dark">

{{/* Generator */}}
{{ hugo.Generator }}

{{/* Canonical */}}
{{ partial "head/canonical.html" . }}

{{/* Open Graph */}}
{{ partial "head/opengraph.html" . }}

{{/* Twitter Card */}}
{{ partial "head/twitter-card.html" . }}

{{/* Hreflang */}}
{{ if .IsTranslated }}
  {{ partial "head/hreflang.html" . }}
{{ end }}

{{/* Structured Data */}}
{{ if .IsHome }}
  {{ partial "structured-data/website.html" . }}
  {{ partial "structured-data/organization.html" . }}
{{ else if .IsPage }}
  {{ partial "structured-data/article.html" . }}
{{ end }}
{{ partial "structured-data/breadcrumb.html" . }}

{{/* Favicon */}}
<link rel="icon" href="/favicon.ico" sizes="32x32">
<link rel="icon" href="/icon.svg" type="image/svg+xml">
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
<link rel="manifest" href="/site.webmanifest">

{{/* RSS Feed */}}
{{ range .AlternativeOutputFormats }}
  {{ printf `<link rel="%s" type="%s" href="%s" title="%s">` .Rel .MediaType.Type .Permalink (printf "%s - %s" $.Site.Title .Name) | safeHTML }}
{{ end }}

{{/* Preconnect */}}
{{ range .Site.Params.preconnect }}
  <link rel="preconnect" href="{{ . }}" crossorigin>
{{ end }}

{{/* DNS Prefetch */}}
{{ range .Site.Params.dns_prefetch }}
  <link rel="dns-prefetch" href="{{ . }}">
{{ end }}

{{/* Verification */}}
{{ with .Site.Params.google_site_verification }}
  <meta name="google-site-verification" content="{{ . }}">
{{ end }}
{{ with .Site.Params.bing_site_verification }}
  <meta name="msvalidate.01" content="{{ . }}">
{{ end }}

{{/* Stylesheets */}}
{{ $styles := resources.Get "css/main.css" | minify | fingerprint }}
<link rel="stylesheet" href="{{ $styles.Permalink }}" integrity="{{ $styles.Data.Integrity }}">
```

### Site Configuration

**config.toml:**

```toml
baseURL = "https://example.com"
title = "Hugo Best Practices"
languageCode = "en-us"

[params]
  description = "Learn to build fast static sites with Hugo"
  author = "Hugo Team"
  theme_color = "#1a1a2e"
  theme_color_dark = "#0d0d1a"

  # Social
  twitter_site = "hugobest"

  # Preconnect
  preconnect = [
    "https://fonts.googleapis.com",
    "https://fonts.gstatic.com"
  ]

  # DNS Prefetch
  dns_prefetch = [
    "https://analytics.example.com"
  ]

  # Verification
  google_site_verification = "your-code"
  bing_site_verification = "your-code"
```

## Meta Tag Order

### Recommended Order in `<head>`

```html
<head>
  <!-- 1. Character encoding (MUST be first) -->
  <meta charset="utf-8">

  <!-- 2. Viewport -->
  <meta name="viewport" content="width=device-width, initial-scale=1">

  <!-- 3. Title -->
  <title>Page Title | Brand</title>

  <!-- 4. SEO meta tags -->
  <meta name="description" content="...">
  <meta name="author" content="...">
  <meta name="robots" content="...">

  <!-- 5. Canonical -->
  <link rel="canonical" href="...">

  <!-- 6. Social media -->
  <meta property="og:..." content="...">
  <meta name="twitter:..." content="...">

  <!-- 7. Hreflang (multilingual) -->
  <link rel="alternate" hreflang="..." href="...">

  <!-- 8. Structured data -->
  <script type="application/ld+json">...</script>

  <!-- 9. Favicon and manifest -->
  <link rel="icon" href="...">
  <link rel="manifest" href="...">

  <!-- 10. Performance hints -->
  <link rel="preconnect" href="...">
  <link rel="dns-prefetch" href="...">
  <link rel="preload" href="...">

  <!-- 11. Stylesheets -->
  <link rel="stylesheet" href="...">

  <!-- 12. RSS/Atom feeds -->
  <link rel="alternate" type="application/rss+xml" href="...">

  <!-- 13. Verification tags -->
  <meta name="google-site-verification" content="...">

  <!-- 14. Theme and display -->
  <meta name="theme-color" content="...">
  <meta name="color-scheme" content="...">
</head>
```

**Why this order?**
- charset must be in first 1024 bytes
- Viewport affects initial render
- SEO tags loaded early for crawlers
- Performance hints before resources

## Best Practices

### General

**✅ DO:**
- Include charset as first element
- Include viewport for responsive design
- Provide unique title and description per page
- Use canonical URLs
- Include Open Graph and Twitter Card tags
- Add structured data (JSON-LD)
- Use preconnect for external origins
- Include favicon
- Set theme-color for mobile browsers

**❌ DON'T:**
- Include meta keywords (deprecated)
- Use duplicate meta tags
- Skip meta description
- Forget canonical URL
- Use http-equiv for things HTTP headers handle better
- Include unnecessary meta tags
- Hardcode verification tags (use config)

### Performance

**✅ DO:**
- Minimize number of meta tags
- Use preconnect for critical external origins
- Preload critical fonts/CSS
- Place meta charset first
- Use Hugo Pipes for CSS (fingerprinting, minification)

**❌ DON'T:**
- Preconnect to too many origins (3-5 max)
- Preload non-critical resources
- Include large inline scripts in head
- Block rendering with synchronous scripts

## Guidelines

### Essential

**Every page must have:**
- `<meta charset="utf-8">`
- `<meta name="viewport">`
- `<title>`
- `<meta name="description">`
- `<link rel="canonical">`
- Favicon

### Recommended

**For better SEO and social sharing:**
- Open Graph tags
- Twitter Card tags
- Structured data (JSON-LD)
- `<meta name="robots">`
- `<meta name="author">`
- Theme color
- RSS feed link
- Hreflang (multilingual sites)

### Advanced

**For maximum optimization:**
- Preconnect and DNS prefetch
- Preload critical resources
- Verification tags
- Color scheme support
- Referrer policy
- Security meta tags

## Benefits

Complete SEO. All search engine signals properly configured.

Social Sharing. Rich previews on all platforms.

Performance. Resource hints improve loading speed.

Accessibility. Proper viewport and color scheme support.

Security. Referrer policy and content security.

## Related

- [meta-descriptions-titles.md](./meta-descriptions-titles.md) - Title and description optimization
- [open-graph-meta-tags.md](./open-graph-meta-tags.md) - Open Graph Protocol
- [twitter-card-types.md](./twitter-card-types.md) - Twitter Card tags
- [canonical-urls.md](./canonical-urls.md) - Canonical URL specification
- [hreflang-tags.md](./hreflang-tags.md) - Multilingual meta tags
- [structured-data-schema-org.md](./structured-data-schema-org.md) - JSON-LD structured data
