# Canonical URLs

Prevent duplicate content penalties. Specify preferred URL version. SEO consolidation. Self-referencing canonicals.

## Principle

Tell search engines which URL is the authoritative version of a page. Consolidate ranking signals. Avoid duplicate content issues.

## What is a Canonical URL?

**Canonical URL:** The preferred version of a web page when multiple URLs show the same or similar content

**Purpose:** Inform search engines which URL should be indexed and ranked

**Format:** `<link rel="canonical">` tag in HTML `<head>`

**Specification:** https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls

## Basic Canonical Tag

```html
<link rel="canonical" href="https://example.com/page">
```

**Placement:** In the `<head>` section of HTML

**Always use absolute URLs** (full https:// URL, not relative paths)

## Common Use Cases

### 1. Self-Referencing Canonical

**Every page should have a canonical tag pointing to itself.**

```html
<!-- On page: https://example.com/blog/post-title -->
<link rel="canonical" href="https://example.com/blog/post-title">
```

**Why?**
- Protects against accidental query parameters (e.g., `?utm_source=twitter`)
- Ensures consistent URL form
- Best practice even for unique pages

### 2. WWW vs Non-WWW

**Problem:** `https://example.com` and `https://www.example.com` are different URLs

```html
<!-- If you prefer non-www, ALL pages should have: -->
<link rel="canonical" href="https://example.com/page">

<!-- If you prefer www, ALL pages should have: -->
<link rel="canonical" href="https://www.example.com/page">
```

**Also configure:** Server-level redirects (301) from non-preferred to preferred version

### 3. HTTP vs HTTPS

**Problem:** `http://example.com` and `https://example.com` are different

```html
<!-- Always use HTTPS version -->
<link rel="canonical" href="https://example.com/page">
```

**Also configure:** 301 redirect from HTTP to HTTPS

### 4. Trailing Slash

**Problem:** `/page` vs `/page/` are different URLs

```html
<!-- Choose one consistently -->
<link rel="canonical" href="https://example.com/page/">

<!-- Or without trailing slash -->
<link rel="canonical" href="https://example.com/page">
```

**Hugo default:** Usually with trailing slash for sections, without for pages

### 5. Query Parameters

**Problem:** URL with tracking parameters is different from base URL

```html
<!-- Original URL: https://example.com/page?utm_source=twitter&ref=123 -->
<!-- Canonical points to clean URL: -->
<link rel="canonical" href="https://example.com/page">
```

**Result:** Search engines index the clean URL, not the parameter-heavy version

### 6. Paginated Content

**Problem:** Multiple pages of results (page 1, 2, 3...)

```html
<!-- On page 1: https://example.com/blog -->
<link rel="canonical" href="https://example.com/blog">

<!-- On page 2: https://example.com/blog/page/2 -->
<link rel="canonical" href="https://example.com/blog/page/2">

<!-- Each page is self-referencing canonical -->
```

**Also use:** `rel="prev"` and `rel="next"` (deprecated by Google but still useful)

```html
<!-- On page 2 -->
<link rel="prev" href="https://example.com/blog">
<link rel="canonical" href="https://example.com/blog/page/2">
<link rel="next" href="https://example.com/blog/page/3">
```

### 7. Duplicate or Similar Content

**Problem:** Same content accessible via multiple URLs

```html
<!-- Product available in multiple categories -->
<!-- URL 1: https://example.com/electronics/laptop-xyz -->
<!-- URL 2: https://example.com/computers/laptop-xyz -->
<!-- URL 3: https://example.com/deals/laptop-xyz -->

<!-- All three should have the same canonical: -->
<link rel="canonical" href="https://example.com/products/laptop-xyz">
```

## Hugo Implementation

### Basic Self-Referencing Canonical

```go-html-template
{{/* layouts/partials/head.html */}}

<link rel="canonical" href="{{ .Permalink }}">
```

**Hugo's `.Permalink`** returns the absolute URL of the page.

### With Preferred Domain Override

```go-html-template
{{/* layouts/partials/head.html */}}

{{- $canonicalURL := .Permalink }}

{{- if .Params.canonical }}
  {{/* Allow front matter override */}}
  {{- $canonicalURL = .Params.canonical | absURL }}
{{- end }}

<link rel="canonical" href="{{ $canonicalURL }}">
```

**Front matter override:**

```yaml
---
title: "My Post"
canonical: "https://example.com/preferred-url"
---
```

### Enforce HTTPS and WWW Preference

```go-html-template
{{/* layouts/partials/canonical.html */}}

{{- $url := .Permalink }}

{{/* Ensure HTTPS */}}
{{- $url = replace $url "http://" "https://" }}

{{/* Enforce www (or remove it) */}}
{{- if .Site.Params.preferWWW }}
  {{- $url = replace $url "https://example.com" "https://www.example.com" }}
{{- else }}
  {{- $url = replace $url "https://www.example.com" "https://example.com" }}
{{- end }}

<link rel="canonical" href="{{ $url }}">
```

**Configuration (config.toml):**

```toml
baseURL = "https://example.com/"  # Without www

[params]
preferWWW = false  # Set to true if you prefer www
```

### Handle Pagination

```go-html-template
{{/* layouts/_default/list.html */}}

<head>
  <!-- Self-referencing canonical for each page -->
  <link rel="canonical" href="{{ .Paginator.URL | absURL }}">

  {{- if .Paginator.HasPrev }}
  <link rel="prev" href="{{ .Paginator.Prev.URL | absURL }}">
  {{- end }}

  {{- if .Paginator.HasNext }}
  <link rel="next" href="{{ .Paginator.Next.URL | absURL }}">
  {{- end }}
</head>
```

## Canonical + Open Graph

**Both should point to the same URL:**

```html
<!-- Canonical -->
<link rel="canonical" href="https://example.com/page">

<!-- Open Graph -->
<meta property="og:url" content="https://example.com/page">
```

**Hugo template:**

```go-html-template
{{- $canonicalURL := .Permalink }}

<!-- Canonical -->
<link rel="canonical" href="{{ $canonicalURL }}">

<!-- Open Graph -->
<meta property="og:url" content="{{ $canonicalURL }}">
```

## Testing Canonicals

### Manual Verification

```bash
# Check canonical tag on a page
curl https://example.com/page | grep "canonical"

# Extract canonical URL
curl https://example.com/page | grep -o '<link rel="canonical" href="[^"]*"'
```

### Google Search Console

**URL Inspection Tool:**
1. Go to Search Console
2. Enter URL to inspect
3. Check "Canonicalization" section
4. Verify Google selected the correct canonical

**Shows:**
- User-declared canonical (your `<link rel="canonical">`)
- Google-selected canonical (what Google actually uses)
- Whether they match

### Browser DevTools

1. Open DevTools (F12)
2. Go to Elements tab
3. Search for "canonical" in `<head>`
4. Verify URL is correct and absolute

## Common Mistakes

❌ **Relative URLs:**
```html
<!-- Wrong -->
<link rel="canonical" href="/page">

<!-- Correct -->
<link rel="canonical" href="https://example.com/page">
```

❌ **HTTP instead of HTTPS:**
```html
<!-- Wrong (if site is HTTPS) -->
<link rel="canonical" href="http://example.com/page">

<!-- Correct -->
<link rel="canonical" href="https://example.com/page">
```

❌ **Multiple canonical tags:**
```html
<!-- Wrong - only one canonical per page -->
<link rel="canonical" href="https://example.com/page1">
<link rel="canonical" href="https://example.com/page2">

<!-- Correct - single canonical -->
<link rel="canonical" href="https://example.com/page">
```

❌ **Canonical pointing to different content:**
```html
<!-- Wrong - canonical should point to same/similar content -->
<!-- Page about cats -->
<link rel="canonical" href="https://example.com/dogs">

<!-- Correct - canonical points to preferred version of THIS content -->
<link rel="canonical" href="https://example.com/cats">
```

❌ **Mismatch with OG URL:**
```html
<!-- Wrong - canonical and og:url should match -->
<link rel="canonical" href="https://example.com/page1">
<meta property="og:url" content="https://example.com/page2">

<!-- Correct - both point to same URL -->
<link rel="canonical" href="https://example.com/page">
<meta property="og:url" content="https://example.com/page">
```

❌ **Missing canonical on paginated pages:**
```html
<!-- Wrong - page 2 points to page 1 as canonical -->
<!-- On https://example.com/blog/page/2 -->
<link rel="canonical" href="https://example.com/blog">

<!-- Correct - self-referencing canonical -->
<link rel="canonical" href="https://example.com/blog/page/2">
```

## Canonical vs 301 Redirect

**When to use canonical:**
- Page is accessible to users (not redirected)
- You want search engines to consolidate signals
- Content is the same or very similar

**When to use 301 redirect:**
- Page has permanently moved
- You want to redirect users AND search engines
- Old URL should not be accessible

**Example:**

```html
<!-- URL with parameters: example.com/page?ref=123 -->
<!-- Show to users, but tell search engines to index clean version -->
<link rel="canonical" href="https://example.com/page">

<!-- vs -->

<!-- Old domain: oldsite.com/page -->
<!-- Redirect users and search engines to new domain -->
<!-- 301 Redirect: oldsite.com/page → newsite.com/page -->
```

## Canonical for Hugo Aliases

**Hugo aliases** create redirects from old URLs to new URLs.

```yaml
---
title: "My Post"
url: "/new-url"
aliases:
  - /old-url
  - /another-old-url
---
```

**Hugo generates HTML redirect pages** for aliases pointing to the new URL.

**Canonical on alias pages:**

Hugo automatically adds canonical to alias pages:

```html
<!-- /old-url/index.html (generated by Hugo) -->
<link rel="canonical" href="https://example.com/new-url">
<meta http-equiv="refresh" content="0; url=https://example.com/new-url">
```

## Cross-Domain Canonicals

**Scenario:** Content is published on multiple domains (e.g., guest posts)

```html
<!-- On partner site: partner.com/article -->
<!-- Point canonical to original on your site -->
<link rel="canonical" href="https://example.com/article">
```

**Result:** Search engines attribute the content to your site, not the partner site.

**Use carefully:** Only if the partner agrees and content is truly identical.

## Canonical for AMP Pages

**Problem:** AMP version and regular HTML version are separate URLs

```html
<!-- On regular page: https://example.com/article -->
<link rel="canonical" href="https://example.com/article">
<link rel="amphtml" href="https://example.com/article/amp">

<!-- On AMP page: https://example.com/article/amp -->
<link rel="canonical" href="https://example.com/article">
```

**Result:** Regular page is canonical, AMP page references it.

## Best Practices

**Always include canonical:**
- Every page should have a canonical tag
- Even unique pages (self-referencing)
- Protects against unexpected duplicates

**Use absolute URLs:**
- Full https://example.com/page format
- Never relative paths (/page)

**Consistency:**
- Match Open Graph `og:url`
- Match sitemap.xml URLs
- Match internal links where possible

**Self-referencing default:**
- Each page's canonical points to itself
- Override only when needed (true duplicates)

**Test regularly:**
- Google Search Console
- Manual inspection
- Automated SEO audits

## Hugo Complete Example

```go-html-template
{{/* layouts/partials/meta/canonical.html */}}

{{- $canonical := .Permalink }}

{{/* Front matter override */}}
{{- if .Params.canonical }}
  {{- $canonical = .Params.canonical | absURL }}
{{- end }}

{{/* Ensure HTTPS */}}
{{- $canonical = replace $canonical "http://" "https://" }}

{{/* Normalize trailing slash */}}
{{- if not (hasPrefix $canonical (printf "%s/" $canonical)) }}
  {{- if .Site.Params.canonicalTrailingSlash }}
    {{- $canonical = printf "%s/" $canonical }}
  {{- end }}
{{- end }}

<!-- Canonical URL -->
<link rel="canonical" href="{{ $canonical }}">

<!-- Open Graph URL (should match canonical) -->
<meta property="og:url" content="{{ $canonical }}">
```

**Configuration (config.toml):**

```toml
baseURL = "https://example.com/"

[params]
canonicalTrailingSlash = false  # true to enforce trailing slash
```

## Guidelines

**Required:**
- `<link rel="canonical">` in `<head>`
- Absolute URL (https://)
- Points to authoritative version

**Recommended:**
- Self-referencing on all pages
- Match `og:url` value
- Consistent with sitemap.xml
- Test in Search Console

**Best Practices:**
- Use `.Permalink` in Hugo
- Allow front matter override
- Normalize www/non-www
- Enforce HTTPS

## Benefits

SEO. Consolidates ranking signals.

Clean. Avoids duplicate content penalties.

Flexible. Allows parameter variations while preserving SEO.

Standard. Works across all search engines.

## Related

- [hreflang-tags.md](./hreflang-tags.md) - Multi-language alternatives
- [meta-tags-seo.md](./meta-tags-seo.md) - HTML meta tags
- [sitemap-xml-advanced.md](./sitemap-xml-advanced.md) - XML sitemaps
- [open-graph-protocol.md](./open-graph-protocol.md) - og:url property
