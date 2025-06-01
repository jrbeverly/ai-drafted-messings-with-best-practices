# Canonical URLs

Canonical URL tags. Duplicate content prevention. Self-referencing canonicals. Cross-domain canonicals. SEO best practices.

## Principle

Use canonical URLs to indicate the preferred version of a page when duplicates exist. Prevent duplicate content penalties. Consolidate ranking signals to the canonical version. Implement self-referencing canonicals for all pages.

## What are Canonical URLs?

**Canonical URL:** The preferred URL for a piece of content

**Purpose:** Tell search engines which URL to index when duplicates exist

**Specified via:** `<link rel="canonical">` tag in HTML `<head>`

**Example:**

```html
<link rel="canonical" href="https://example.com/blog/post/">
```

**Specification:** https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls

## Why Use Canonical URLs?

### Duplicate Content Issues

**Common duplicate scenarios:**

1. **URL variations:**
   - https://example.com/page
   - https://example.com/page/
   - https://example.com/page?utm_source=twitter
   - https://www.example.com/page
   - http://example.com/page

2. **Pagination:**
   - https://example.com/blog/
   - https://example.com/blog/page/2/
   - https://example.com/blog/page/3/

3. **Print/mobile versions:**
   - https://example.com/article
   - https://example.com/article?print=true
   - https://m.example.com/article

4. **Syndicated content:**
   - Original: https://yourblog.com/article
   - Syndicated: https://medium.com/your-article

**Without canonical tags:**
- Search engines may index wrong version
- Link equity split across duplicates
- Diluted ranking signals
- Potential duplicate content penalty

**With canonical tags:**
- Search engines index preferred URL
- Link equity consolidated
- Clear preferred version
- No duplicate content issues

## Canonical Tag Syntax

### Basic Syntax

```html
<link rel="canonical" href="https://example.com/preferred-url/">
```

**Requirements:**
- Must be in `<head>` section
- Must use absolute URL (not relative)
- Must use HTTPS (if site supports it)
- Should include trailing slash (if site uses it)

### Self-Referencing Canonical

**Every page should have canonical tag pointing to itself:**

```html
<!-- Page: https://example.com/blog/post/ -->
<head>
  <link rel="canonical" href="https://example.com/blog/post/">
</head>
```

**Why:**
- Prevents URL parameter issues
- Clarifies preferred URL format
- Best practice recommended by Google
- Protects against scraping with URL parameters

### Cross-Domain Canonical

**Point to content on different domain:**

```html
<!-- Syndicated article on Medium -->
<head>
  <link rel="canonical" href="https://yourblog.com/original-article/">
</head>
```

**Use cases:**
- Syndicated content (Medium, LinkedIn)
- Content republishing
- Partner sites
- Product descriptions from manufacturer

## Hugo Implementation

### Automatic Canonical URLs

**Hugo includes canonical URLs automatically in default templates.**

**Default behavior:**
- Hugo adds `<link rel="canonical">` to all pages
- Uses `.Permalink` (absolute URL)
- Self-referencing by default

**Default template (built-in):**

```go-html-template
<link rel="canonical" href="{{ .Permalink }}">
```

### Custom Canonical Template

**Override or customize:**

**layouts/partials/head/canonical.html:**

```go-html-template
{{/* Canonical URL */}}
{{ if .Params.canonical }}
  {{/* Custom canonical from front matter */}}
  <link rel="canonical" href="{{ .Params.canonical }}">
{{ else if .IsHome }}
  {{/* Homepage */}}
  <link rel="canonical" href="{{ .Site.BaseURL }}">
{{ else }}
  {{/* Self-referencing canonical */}}
  <link rel="canonical" href="{{ .Permalink }}">
{{ end }}
```

**Include in layout:**

**layouts/_default/baseof.html:**

```go-html-template
<head>
  <meta charset="utf-8">
  <title>{{ .Title }}</title>

  {{/* Canonical URL */}}
  {{ partial "head/canonical.html" . }}

  {{/* Other head elements */}}
</head>
```

### Front Matter Override

**Set custom canonical in front matter:**

```yaml
---
title: "Syndicated Article"
canonical: "https://originalblog.com/article/"
---
```

**Hugo template handles it:**

```go-html-template
{{ if .Params.canonical }}
  <link rel="canonical" href="{{ .Params.canonical }}">
{{ else }}
  <link rel="canonical" href="{{ .Permalink }}">
{{ end }}
```

### Pagination Canonical

**For paginated lists, point to first page:**

```go-html-template
{{ if .Paginator }}
  {{ if eq .Paginator.PageNumber 1 }}
    {{/* First page - self-referencing */}}
    <link rel="canonical" href="{{ .Permalink }}">
  {{ else }}
    {{/* Subsequent pages - point to first page */}}
    <link rel="canonical" href="{{ .Paginator.First.URL | absURL }}">
  {{ end }}
{{ else }}
  {{/* Non-paginated page */}}
  <link rel="canonical" href="{{ .Permalink }}">
{{ end }}
```

**Example:**
- `/blog/` → canonical: `https://example.com/blog/`
- `/blog/page/2/` → canonical: `https://example.com/blog/`
- `/blog/page/3/` → canonical: `https://example.com/blog/`

### Language-Specific Canonicals

**For multilingual sites:**

```go-html-template
{{/* Each language version has its own canonical */}}
<link rel="canonical" href="{{ .Permalink }}">

{{/* Plus hreflang tags for alternate languages */}}
{{ range .Translations }}
  <link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">
{{ end }}
```

**Don't point all languages to one canonical** (each language is unique content).

### Dynamic Canonical Based on Parameters

**Remove URL parameters from canonical:**

```go-html-template
{{ $canonical := .Permalink }}

{{/* Strip query parameters for canonical */}}
{{ if strings.Contains $canonical "?" }}
  {{ $canonical = (split $canonical "?")._0 }}
{{ end }}

<link rel="canonical" href="{{ $canonical }}">
```

**Example:**
- URL: `https://example.com/blog/post/?utm_source=twitter`
- Canonical: `https://example.com/blog/post/`

## Configuration

### Site-wide Settings

**config.toml:**

```toml
baseURL = "https://example.com"

# Ensure canonical URLs use HTTPS
[params]
  canonicalBaseURL = "https://example.com"
```

### Ensure Absolute URLs

**Hugo uses `.Permalink` which is always absolute:**

```go-html-template
{{ .Permalink }}
<!-- Output: https://example.com/blog/post/ -->
```

**Never use `.RelPermalink` for canonical:**

```go-html-template
{{/* ❌ WRONG - relative URL */}}
<link rel="canonical" href="{{ .RelPermalink }}">

{{/* ✅ CORRECT - absolute URL */}}
<link rel="canonical" href="{{ .Permalink }}">
```

## Common Use Cases

### Self-Referencing Canonical

**Every page points to itself:**

```html
<!-- Page: https://example.com/blog/hugo-tips/ -->
<head>
  <link rel="canonical" href="https://example.com/blog/hugo-tips/">
</head>
```

**Why:**
- Prevents URL parameter pollution
- Clarifies preferred URL format (trailing slash, HTTPS, www/non-www)
- Best practice

### Pagination

**All paginated pages point to first page:**

```html
<!-- Page 1: https://example.com/blog/ -->
<link rel="canonical" href="https://example.com/blog/">

<!-- Page 2: https://example.com/blog/page/2/ -->
<link rel="canonical" href="https://example.com/blog/">

<!-- Page 3: https://example.com/blog/page/3/ -->
<link rel="canonical" href="https://example.com/blog/">
```

**Alternative (Google's preference):**
- Each paginated page is self-referencing canonical
- Use `rel="prev"` and `rel="next"` links (deprecated but still useful)

```html
<!-- Page 2 -->
<link rel="canonical" href="https://example.com/blog/page/2/">
<link rel="prev" href="https://example.com/blog/">
<link rel="next" href="https://example.com/blog/page/3/">
```

### Syndicated Content

**Republished article points to original:**

```html
<!-- Syndicated on Medium -->
<head>
  <link rel="canonical" href="https://yourblog.com/original-article/">
</head>
```

**Front matter:**

```yaml
---
title: "Republished Article"
canonical: "https://originalblog.com/article/"
---
```

### AMP Pages

**AMP version points to canonical HTML version:**

```html
<!-- AMP page: https://example.com/blog/post/amp/ -->
<head>
  <link rel="canonical" href="https://example.com/blog/post/">
</head>
```

**Canonical HTML page links to AMP:**

```html
<!-- HTML page: https://example.com/blog/post/ -->
<head>
  <link rel="canonical" href="https://example.com/blog/post/">
  <link rel="amphtml" href="https://example.com/blog/post/amp/">
</head>
```

### WWW vs Non-WWW

**Choose one and stick with it:**

**Option 1: Non-WWW (recommended for modern sites)**

```toml
baseURL = "https://example.com"
```

**All canonicals:**

```html
<link rel="canonical" href="https://example.com/page/">
```

**Redirect www to non-www:**

```
# Netlify _redirects
https://www.example.com/* https://example.com/:splat 301!
```

**Option 2: WWW**

```toml
baseURL = "https://www.example.com"
```

**All canonicals:**

```html
<link rel="canonical" href="https://www.example.com/page/">
```

### HTTP vs HTTPS

**Always use HTTPS in canonical (if supported):**

```html
<!-- ✅ CORRECT -->
<link rel="canonical" href="https://example.com/page/">

<!-- ❌ WRONG -->
<link rel="canonical" href="http://example.com/page/">
```

**Redirect HTTP to HTTPS:**

```
# Netlify _redirects
http://example.com/* https://example.com/:splat 301!
```

### Trailing Slash Consistency

**Choose one format:**

**With trailing slash (common for static sites):**

```html
<link rel="canonical" href="https://example.com/blog/post/">
```

**Hugo default:**

```toml
# Hugo adds trailing slash by default
```

**Without trailing slash:**

```toml
# config.toml
[permalinks]
  posts = "/:year/:month/:filename"  # No trailing slash
```

```html
<link rel="canonical" href="https://example.com/blog/post">
```

**Pick one and be consistent across entire site.**

## Best Practices

### General

**✅ DO:**
- Include canonical tag on every page
- Use self-referencing canonicals
- Use absolute URLs (https://example.com/page/)
- Use HTTPS (if site supports it)
- Be consistent with URL format (trailing slash, www)
- Place in `<head>` section
- Use only one canonical tag per page

**❌ DON'T:**
- Use relative URLs (/page/)
- Use HTTP for canonical on HTTPS site
- Include multiple canonical tags
- Point all pages to homepage
- Change canonical URL after publication
- Use canonical as a redirect (use 301 instead)

### Pagination

**✅ DO:**
- Option 1: Point all pages to first page (simple)
- Option 2: Self-referencing + rel="prev"/rel="next" (Google's preference)
- Be consistent across site

**❌ DON'T:**
- Point to arbitrary page
- Omit canonical on paginated pages
- Mix strategies across site

### Syndication

**✅ DO:**
- Point syndicated content to original
- Wait 2+ weeks before syndicating (let Google index original first)
- Use cross-domain canonical
- Inform syndication partner to add canonical

**❌ DON'T:**
- Syndicate immediately after publishing
- Forget to add canonical to syndicated version
- Point original to syndicated version

### URL Parameters

**✅ DO:**
- Strip tracking parameters from canonical
- Use self-referencing canonical without parameters
- Configure URL parameters in Google Search Console

**❌ DON'T:**
- Include tracking parameters in canonical
- Block parameter URLs in robots.txt (canonicals handle it)

## Testing and Validation

### Check Canonical Tag Exists

```bash
curl -s https://example.com/blog/post/ | grep -i 'rel="canonical"'
```

**Expected output:**

```html
<link rel="canonical" href="https://example.com/blog/post/">
```

### Validate URL Format

**Check:**
- ✅ Absolute URL (starts with https://)
- ✅ HTTPS (not http://)
- ✅ Consistent format (trailing slash)
- ✅ Matches preferred domain (www or non-www)

### Google Search Console

**URL Inspection Tool:**
1. Go to Google Search Console
2. Enter URL
3. Check "Canonical URL" section
4. Verify Google respects your canonical tag

**Possible statuses:**
- ✅ "User-declared canonical" - Your canonical tag
- ✅ "Google-selected canonical" - Google agrees
- ⚠️ "Alternate page with canonical tag" - This is duplicate, points to canonical
- ❌ "Duplicate without user-selected canonical" - Missing canonical tag

### Screaming Frog SEO Spider

**Bulk check canonical tags:**
1. Crawl site with Screaming Frog
2. Go to "Canonicals" tab
3. Check all pages have canonical
4. Verify canonical URLs are correct

### Manual Validation

**Check source code:**

```html
<!-- View page source -->
<head>
  <link rel="canonical" href="https://example.com/blog/post/">
</head>
```

**Validate:**
- Tag is in `<head>`
- URL is absolute
- URL is correct format
- Only one canonical tag

## Common Issues

### Missing Canonical Tag

**Problem:** Page has no canonical tag

**Impact:** Search engines choose canonical arbitrarily

**Solution:**

```go-html-template
{{/* Ensure all pages have canonical */}}
<link rel="canonical" href="{{ .Permalink }}">
```

### Relative URL in Canonical

**Problem:**

```html
<!-- ❌ WRONG -->
<link rel="canonical" href="/blog/post/">
```

**Impact:** Search engines may ignore or misinterpret

**Solution:**

```html
<!-- ✅ CORRECT -->
<link rel="canonical" href="https://example.com/blog/post/">
```

**Hugo fix:**

```go-html-template
{{/* Use .Permalink (always absolute) */}}
<link rel="canonical" href="{{ .Permalink }}">
```

### HTTP Canonical on HTTPS Site

**Problem:**

```html
<!-- ❌ WRONG - site is HTTPS -->
<link rel="canonical" href="http://example.com/page/">
```

**Impact:** Mixed content, search engines may be confused

**Solution:**

```toml
# config.toml
baseURL = "https://example.com"  # Use https://
```

### Multiple Canonical Tags

**Problem:**

```html
<head>
  <link rel="canonical" href="https://example.com/page/">
  <link rel="canonical" href="https://example.com/other-page/">
</head>
```

**Impact:** Search engines ignore all canonical tags

**Solution:**

Ensure only one canonical tag per page:

```go-html-template
{{/* Only one canonical tag */}}
<link rel="canonical" href="{{ .Permalink }}">
```

### Canonical Points to Redirect

**Problem:**

```html
<!-- Canonical points to URL that redirects -->
<link rel="canonical" href="https://example.com/old-url/">
<!-- old-url redirects to new-url -->
```

**Impact:** Inefficient, potential indexing issues

**Solution:**

Point canonical directly to final destination:

```html
<link rel="canonical" href="https://example.com/new-url/">
```

### Canonical Points to 404

**Problem:** Canonical URL returns 404

**Impact:** Search engines may deindex page

**Solution:**

Ensure canonical URL is accessible:

```bash
curl -I https://example.com/canonical-url/
# Should return: HTTP/2 200
```

### Wrong Domain in Canonical

**Problem:**

```html
<!-- Page on example.com -->
<link rel="canonical" href="https://wrongdomain.com/page/">
```

**Impact:** Gives ranking signals to wrong domain

**Solution:**

Use correct domain (self-referencing):

```html
<link rel="canonical" href="https://example.com/page/">
```

## Advanced Techniques

### Canonical for URL Parameters

**Strip parameters automatically:**

```go-html-template
{{ $url := .Permalink }}

{{/* Remove query parameters */}}
{{ if strings.Contains $url "?" }}
  {{ $url = index (split $url "?") 0 }}
{{ end }}

{{/* Remove fragments */}}
{{ if strings.Contains $url "#" }}
  {{ $url = index (split $url "#") 0 }}
{{ end }}

<link rel="canonical" href="{{ $url }}">
```

### Conditional Canonical for Environments

**Different canonical for staging vs production:**

```go-html-template
{{ if eq (getenv "HUGO_ENV") "production" }}
  <link rel="canonical" href="{{ .Permalink }}">
{{ else }}
  {{/* Staging: point to production canonical */}}
  {{ $prodURL := replace .Permalink "staging.example.com" "example.com" }}
  <link rel="canonical" href="{{ $prodURL }}">
{{ end }}
```

### Canonical for Taxonomy Pages

**For tag/category pages:**

```go-html-template
{{ if .IsHome }}
  <link rel="canonical" href="{{ .Site.BaseURL }}">
{{ else if eq .Kind "taxonomy" }}
  {{/* Tag/category pages - self-referencing */}}
  <link rel="canonical" href="{{ .Permalink }}">
{{ else if eq .Kind "term" }}
  {{/* Individual tag page */}}
  <link rel="canonical" href="{{ .Permalink }}">
{{ else }}
  <link rel="canonical" href="{{ .Permalink }}">
{{ end }}
```

## Complete Example

**layouts/partials/head/canonical.html:**

```go-html-template
{{/* Canonical URL */}}
{{ $canonical := "" }}

{{ if .Params.canonical }}
  {{/* Custom canonical from front matter (syndication, etc.) */}}
  {{ $canonical = .Params.canonical }}

{{ else if .IsHome }}
  {{/* Homepage */}}
  {{ $canonical = .Site.BaseURL }}

{{ else if .Paginator }}
  {{/* Pagination - point to first page */}}
  {{ if eq .Paginator.PageNumber 1 }}
    {{ $canonical = .Permalink }}
  {{ else }}
    {{ $canonical = .Paginator.First.URL | absURL }}
  {{ end }}

{{ else }}
  {{/* Regular pages - self-referencing */}}
  {{ $canonical = .Permalink }}

{{ end }}

{{/* Strip query parameters from canonical */}}
{{ if strings.Contains $canonical "?" }}
  {{ $canonical = index (split $canonical "?") 0 }}
{{ end }}

{{/* Output canonical tag */}}
<link rel="canonical" href="{{ $canonical }}">
```

**Usage in baseof.html:**

```go-html-template
<!DOCTYPE html>
<html lang="{{ .Site.Language.Lang }}">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{{ .Title }}</title>

  {{/* Canonical URL */}}
  {{ partial "head/canonical.html" . }}

  {{/* Other head elements */}}
</head>
<body>
  {{ block "main" . }}{{ end }}
</body>
</html>
```

## Guidelines

### Essential

**Every page must have:**
- Canonical tag in `<head>`
- Absolute URL (https://example.com/page/)
- HTTPS protocol (if site supports it)
- Consistent URL format

**Minimum implementation:**

```html
<link rel="canonical" href="{{ .Permalink }}">
```

### Recommended

**For better SEO:**
- Self-referencing canonicals on all pages
- Strip URL parameters from canonical
- Pagination canonical strategy
- Test with Google Search Console
- Consistent trailing slash policy

### Advanced

**For maximum control:**
- Custom canonicals via front matter
- Cross-domain canonicals for syndication
- Environment-specific canonicals
- Automatic parameter stripping
- Integration with hreflang for multilingual

## Benefits

Duplicate Prevention. Avoid duplicate content penalties.

Link Equity. Consolidate ranking signals to preferred URL.

Clarity. Tell search engines which URL to index.

Flexibility. Support syndication and content republishing.

Control. Choose exact URL format for indexing.

## Related

- [hreflang-tags.md](./hreflang-tags.md) - Multilingual canonical handling
- [sitemaps-robots-txt.md](./sitemaps-robots-txt.md) - Include canonical URLs in sitemap
- [meta-descriptions-titles.md](./meta-descriptions-titles.md) - Complete meta tag strategy
- [structured-data-schema-org.md](./structured-data-schema-org.md) - Use canonical URL in schema
- [../../01-hugo-basics/hugo-content-management.md](../01-hugo-basics/hugo-content-management.md) - Content organization
