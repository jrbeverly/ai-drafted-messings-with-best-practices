# Meta Tags (SEO)

HTML meta tags for search engines. Description, keywords, viewport, robots. Essential SEO metadata.

## Principle

Provide metadata about your web page to browsers and search engines. Control indexing, display, and social sharing. Optimize for search visibility and user experience.

## What are Meta Tags?

**Meta tags:** HTML elements in `<head>` that provide metadata about the web page

**Format:** `<meta name="..." content="...">`

**Purpose:**
- Describe page content (SEO)
- Control search engine behavior
- Configure viewport (mobile)
- Social media sharing (Open Graph, Twitter Cards)
- Browser behavior

**Not visible:** Meta tags don't appear in page content (only in HTML source)

## Essential Meta Tags

### 1. Charset (Required)

```html
<meta charset="UTF-8">
```

**Purpose:** Specifies character encoding (always UTF-8)

**Placement:** First tag in `<head>` (before title)

### 2. Viewport (Required for Mobile)

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

**Purpose:** Responsive design configuration

**Properties:**
- `width=device-width`: Match screen width
- `initial-scale=1.0`: No zoom on load
- `maximum-scale=5.0`: Allow zoom (accessibility)
- `user-scalable=yes`: Allow pinch-zoom (default)

**Never disable zoom:** `user-scalable=no` harms accessibility

### 3. Title (Required, technically not a meta tag)

```html
<title>Page Title - Site Name</title>
```

**Purpose:** Page title shown in search results and browser tab

**Length:** 50-60 characters (Google truncates ~600px width)

**Format:** `Page Title | Site Name` or `Page Title - Site Name`

### 4. Description (Highly Recommended)

```html
<meta name="description" content="Compelling description of the page content that appears in search results.">
```

**Purpose:** Snippet text in search engine results

**Length:** 150-160 characters (Google truncates ~920px width)

**Best practices:**
- Clear, compelling summary
- Include target keywords naturally
- Unique for each page
- Action-oriented when appropriate

### 5. Canonical URL (Highly Recommended)

```html
<link rel="canonical" href="https://example.com/page">
```

**Purpose:** Specify preferred URL version

**See:** [canonical-urls.md](./canonical-urls.md)

## SEO Meta Tags

### Robots

```html
<!-- Allow indexing and following links (default) -->
<meta name="robots" content="index, follow">

<!-- Prevent indexing -->
<meta name="robots" content="noindex, nofollow">

<!-- Index but don't follow links -->
<meta name="robots" content="index, nofollow">

<!-- Advanced directives -->
<meta name="robots" content="index, follow, max-snippet:-1, max-image-preview:large, max-video-preview:-1">
```

**Common values:**
- `index` / `noindex`: Allow/prevent search indexing
- `follow` / `nofollow`: Follow/don't follow links on page
- `noarchive`: Don't show cached version in search
- `nosnippet`: Don't show text snippet in search results
- `max-snippet:N`: Max characters in snippet (-1 = no limit)
- `max-image-preview:large`: Allow large image previews
- `max-video-preview:N`: Max seconds of video preview (-1 = no limit)

**Specific bots:**

```html
<!-- Google-specific -->
<meta name="googlebot" content="index, follow">

<!-- Bing-specific -->
<meta name="bingbot" content="index, follow">
```

### Keywords (Obsolete)

```html
<!-- ❌ Don't use - ignored by Google since 2009 -->
<meta name="keywords" content="web, development, SEO">
```

**Status:** Ignored by major search engines (Google, Bing)

**History:** Abused for keyword stuffing, no longer used

### Author

```html
<meta name="author" content="Jane Doe">
```

**Purpose:** Specify page author

**SEO impact:** Minimal (use structured data instead)

### Theme Color (Mobile Browsers)

```html
<meta name="theme-color" content="#2196F3">
```

**Purpose:** Browser UI color on mobile (address bar, etc.)

**Supports:** Chrome, Edge, Safari on mobile

### Referrer Policy

```html
<meta name="referrer" content="strict-origin-when-cross-origin">
```

**Purpose:** Control how much referrer information is sent

**Values:**
- `no-referrer`: Never send referrer
- `no-referrer-when-downgrade`: Send unless HTTPS→HTTP
- `origin`: Send only origin (https://example.com)
- `origin-when-cross-origin`: Full URL for same-origin, origin only for cross-origin
- `strict-origin-when-cross-origin`: Recommended (default in modern browsers)

## Hugo Implementation

### Basic Meta Tags Template

```go-html-template
{{/* layouts/partials/meta.html */}}

<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<title>{{ .Title }} | {{ .Site.Title }}</title>

{{- with .Description }}
<meta name="description" content="{{ . }}">
{{- else }}
<meta name="description" content="{{ .Site.Params.description }}">
{{- end }}

<link rel="canonical" href="{{ .Permalink }}">

{{- with .Params.author }}
<meta name="author" content="{{ . }}">
{{- else }}
<meta name="author" content="{{ .Site.Params.author.name }}">
{{- end }}

{{- if .Params.noindex }}
<meta name="robots" content="noindex, nofollow">
{{- else }}
<meta name="robots" content="index, follow, max-snippet:-1, max-image-preview:large">
{{- end }}

<meta name="theme-color" content="{{ .Site.Params.themeColor | default "#2196F3" }}">
```

### Configuration (config.toml)

```toml
title = "My Website"

[params]
description = "Default site description"
themeColor = "#2196F3"

[params.author]
name = "Jane Doe"
```

### Front Matter Control

```yaml
---
title: "Page Title"
description: "Custom description for this page"
author: "John Smith"
noindex: false  # Set to true to prevent indexing
---
```

## Complete HTML Head Example

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <!-- Charset (first) -->
  <meta charset="UTF-8">

  <!-- Viewport (mobile) -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <!-- Title -->
  <title>Complete Guide to Meta Tags | Example Blog</title>

  <!-- SEO Meta Tags -->
  <meta name="description" content="Learn how to implement essential SEO meta tags for better search engine visibility and user experience.">
  <meta name="author" content="Jane Doe">
  <meta name="robots" content="index, follow, max-snippet:-1, max-image-preview:large">

  <!-- Canonical URL -->
  <link rel="canonical" href="https://example.com/guides/meta-tags">

  <!-- Open Graph (Social Media) -->
  <meta property="og:title" content="Complete Guide to Meta Tags">
  <meta property="og:description" content="Learn how to implement essential SEO meta tags.">
  <meta property="og:image" content="https://example.com/images/meta-tags.jpg">
  <meta property="og:url" content="https://example.com/guides/meta-tags">
  <meta property="og:type" content="article">
  <meta property="og:site_name" content="Example Blog">

  <!-- Twitter Cards -->
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:title" content="Complete Guide to Meta Tags">
  <meta name="twitter:description" content="Learn how to implement essential SEO meta tags.">
  <meta name="twitter:image" content="https://example.com/images/meta-tags.jpg">
  <meta name="twitter:site" content="@example">

  <!-- Favicon -->
  <link rel="icon" href="/favicon.svg" type="image/svg+xml">
  <link rel="apple-touch-icon" href="/apple-touch-icon.png">

  <!-- Theme Color -->
  <meta name="theme-color" content="#2196F3">

  <!-- Referrer Policy -->
  <meta name="referrer" content="strict-origin-when-cross-origin">

  <!-- Stylesheet -->
  <link rel="stylesheet" href="/css/style.css">
</head>
<body>
  <!-- Page content -->
</body>
</html>
```

## Hugo Complete Template

```go-html-template
{{/* layouts/partials/head.html */}}

<head>
  <!-- Charset -->
  <meta charset="UTF-8">

  <!-- Viewport -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <!-- Title -->
  {{- if .IsHome }}
  <title>{{ .Site.Title }} - {{ .Site.Params.tagline }}</title>
  {{- else }}
  <title>{{ .Title }} | {{ .Site.Title }}</title>
  {{- end }}

  <!-- Description -->
  {{- with .Description }}
  <meta name="description" content="{{ . }}">
  {{- else }}
  <meta name="description" content="{{ .Site.Params.description }}">
  {{- end }}

  <!-- Canonical -->
  <link rel="canonical" href="{{ .Permalink }}">

  <!-- Author -->
  {{- with .Params.author }}
  <meta name="author" content="{{ . }}">
  {{- else }}
  <meta name="author" content="{{ .Site.Params.author.name }}">
  {{- end }}

  <!-- Robots -->
  {{- if or .Params.noindex (eq .Kind "taxonomy") }}
  <meta name="robots" content="noindex, nofollow">
  {{- else }}
  <meta name="robots" content="index, follow, max-snippet:-1, max-image-preview:large">
  {{- end }}

  <!-- Open Graph -->
  {{ partial "open-graph.html" . }}

  <!-- Twitter Cards -->
  {{ partial "twitter-cards.html" . }}

  <!-- Hreflang (if multilingual) -->
  {{- if .Site.IsMultiLingual }}
  {{ partial "hreflang.html" . }}
  {{- end }}

  <!-- Favicon -->
  <link rel="icon" href="/favicon.svg" type="image/svg+xml">
  <link rel="icon" href="/favicon.png" type="image/png">
  <link rel="apple-touch-icon" href="/apple-touch-icon.png">

  <!-- Theme Color -->
  <meta name="theme-color" content="{{ .Site.Params.themeColor | default "#FFFFFF" }}">

  <!-- Referrer Policy -->
  <meta name="referrer" content="strict-origin-when-cross-origin">

  <!-- RSS Feed -->
  {{ with .OutputFormats.Get "RSS" }}
  <link rel="alternate" type="application/rss+xml" title="{{ $.Site.Title }}" href="{{ .Permalink }}">
  {{ end }}

  <!-- Structured Data (JSON-LD) -->
  {{ partial "structured-data.html" . }}

  <!-- Stylesheets -->
  {{- range .Site.Params.css }}
  <link rel="stylesheet" href="{{ . | relURL }}">
  {{- end }}
</head>
```

## Meta Tags for Special Cases

### Noindex for Development/Staging

```html
<!-- Prevent indexing of staging site -->
<meta name="robots" content="noindex, nofollow">
```

**Hugo (environment-based):**

```go-html-template
{{- if eq (getenv "HUGO_ENV") "production" }}
<meta name="robots" content="index, follow">
{{- else }}
<meta name="robots" content="noindex, nofollow">
{{- end }}
```

### Noindex for Paginated Pages

```go-html-template
{{- if .Paginator }}
  {{- if gt .Paginator.PageNumber 1 }}
<meta name="robots" content="noindex, follow">
  {{- end }}
{{- end }}
```

### Prevent Caching

```html
<!-- Prevent browser caching (use sparingly) -->
<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
<meta http-equiv="Pragma" content="no-cache">
<meta http-equiv="Expires" content="0">
```

**Note:** Use HTTP headers for cache control instead (more reliable)

### IE Compatibility Mode (Legacy)

```html
<!-- Force latest IE rendering engine (legacy, rarely needed) -->
<meta http-equiv="X-UA-Compatible" content="IE=edge">
```

**Modern approach:** Don't include (IE is obsolete)

## Testing Meta Tags

### Browser DevTools

1. Open DevTools (F12)
2. Elements tab
3. View `<head>` section
4. Verify all meta tags present

### View Source

Right-click → "View Page Source" → Search for `<meta`

### Command Line

```bash
# Extract all meta tags
curl https://example.com | grep "<meta"

# Check specific meta tag
curl https://example.com | grep 'name="description"'

# Check title
curl https://example.com | grep "<title>"
```

### Online Tools

**Google Rich Results Test:** https://search.google.com/test/rich-results
**Facebook Sharing Debugger:** https://developers.facebook.com/tools/debug/
**Twitter Card Validator:** https://cards-dev.twitter.com/validator
**SEO Meta Tag Checker:** https://www.seoptimer.com/meta-tag-checker

### Google Search Console

**URL Inspection Tool:**
1. Enter URL
2. View "Coverage" → "Indexing"
3. Check detected meta tags

## Common Mistakes

❌ **Missing viewport:**
```html
<!-- Wrong - no viewport = poor mobile UX -->
<head>
  <title>Page Title</title>
</head>

<!-- Correct -->
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Page Title</title>
</head>
```

❌ **Duplicate meta tags:**
```html
<!-- Wrong - multiple descriptions -->
<meta name="description" content="First description">
<meta name="description" content="Second description">

<!-- Correct - single description -->
<meta name="description" content="The description">
```

❌ **Description too long:**
```html
<!-- Wrong - 300 characters, will be truncated -->
<meta name="description" content="This is a very long description that goes on and on and provides way too much information that will definitely be truncated by Google because it exceeds the recommended 160 character limit and users won't see the full text in search results...">

<!-- Correct - concise, ~150-160 chars -->
<meta name="description" content="Learn essential meta tags for SEO including title, description, canonical, and Open Graph tags for better search visibility.">
```

❌ **Using keywords meta tag:**
```html
<!-- Wrong - ignored by search engines -->
<meta name="keywords" content="SEO, meta tags, HTML">

<!-- Correct - don't include keywords tag -->
```

❌ **Charset not first:**
```html
<!-- Wrong - charset should be first -->
<head>
  <title>Page</title>
  <meta charset="UTF-8">
</head>

<!-- Correct - charset first -->
<head>
  <meta charset="UTF-8">
  <title>Page</title>
</head>
```

❌ **Disabling zoom:**
```html
<!-- Wrong - harms accessibility -->
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0">

<!-- Correct - allow zoom -->
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

## Best Practices

**Required:**
- `<meta charset="UTF-8">` (first tag)
- `<meta name="viewport">` (mobile)
- `<title>` (unique per page)
- `<meta name="description">` (unique per page)
- `<link rel="canonical">` (self-referencing)

**Recommended:**
- `<meta name="robots">` (control indexing)
- `<meta name="theme-color">` (mobile)
- Open Graph tags (social sharing)
- Twitter Card tags (Twitter sharing)
- Favicon links

**Optional:**
- `<meta name="author">` (minimal SEO impact)
- Structured data JSON-LD (for rich results)
- Hreflang tags (multilingual sites)

**Avoid:**
- `<meta name="keywords">` (obsolete)
- Duplicate meta tags
- Disabling zoom (`user-scalable=no`)
- Very long descriptions (>160 chars)

## Meta Tags Priority

**Order in `<head>`:**

1. `<meta charset>` (first)
2. `<meta name="viewport">`
3. `<title>`
4. `<meta name="description">`
5. `<link rel="canonical">`
6. `<meta name="robots">`
7. Open Graph tags
8. Twitter Card tags
9. Other meta tags
10. Stylesheets and scripts

## Hugo Testing

```bash
# Build site
hugo

# Check generated meta tags
grep -r "<meta" public/

# Check specific page
cat public/guides/meta-tags/index.html | grep "<meta"

# Serve locally and inspect
hugo server
# Open http://localhost:1313 and view source
```

## Guidelines

**Essential Meta Tags:**
```html
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Page Title | Site Name</title>
<meta name="description" content="150-160 character description">
<link rel="canonical" href="https://example.com/page">
```

**SEO Optimization:**
- Unique title and description per page
- Include target keywords naturally
- Match description to page content
- Keep titles under 60 characters
- Keep descriptions under 160 characters

**Accessibility:**
- Never disable zoom
- Include `lang` attribute on `<html>`
- Use semantic HTML alongside meta tags

## Benefits

Discoverable. Search engines understand your content.

Clickable. Compelling descriptions increase CTR.

Shareable. Social media cards look professional.

Accessible. Proper viewport enables responsive design.

## Related

- [open-graph-protocol.md](./open-graph-protocol.md) - Social media tags
- [twitter-cards.md](./twitter-cards.md) - Twitter-specific tags
- [canonical-urls.md](./canonical-urls.md) - Canonical link tag
- [structured-data-json-ld.md](./structured-data-json-ld.md) - JSON-LD
- [hreflang-tags.md](./hreflang-tags.md) - Multi-language tags
