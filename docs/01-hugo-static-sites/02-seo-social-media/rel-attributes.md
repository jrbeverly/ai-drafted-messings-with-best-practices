# Rel Attributes

Link relationship attributes. rel="nofollow". rel="noopener". rel="canonical". HTML link semantics. Hugo implementation.

## Principle

Use rel attributes to define the relationship between the current document and linked resources. Implement correct rel values for security, SEO, and semantic meaning. Follow best practices for external links, navigation, and resource hints.

## What are Rel Attributes?

**Rel (relationship):** HTML attribute that specifies the relationship between documents

**Used on:**
- `<link>` elements in `<head>`
- `<a>` anchor elements
- `<area>` elements

**Purpose:**
- Security (noopener, noreferrer)
- SEO (nofollow, canonical, sponsored)
- Navigation (prev, next)
- Resource hints (preconnect, prefetch, preload)
- Semantic relationships (author, license)

## SEO Rel Attributes

### nofollow

**Tells search engines not to pass link equity.**

```html
<a href="https://external.com" rel="nofollow">External Link</a>
```

**When to use:**
- User-generated content (comments, forums)
- Untrusted links
- Paid links (also use rel="sponsored")
- Links you don't want to endorse

### sponsored

**Identifies paid/sponsored links.**

```html
<a href="https://sponsor.com" rel="sponsored">Our Sponsor</a>
```

**When to use:**
- Paid advertisements
- Sponsored content links
- Affiliate links
- Partnership links

### ugc (User Generated Content)

**Identifies links from user-generated content.**

```html
<a href="https://example.com" rel="ugc">User's Link</a>
```

**When to use:**
- Blog comments
- Forum posts
- User profiles
- Community content

### Combining SEO Rel Values

```html
<!-- Paid link in user content -->
<a href="https://example.com" rel="nofollow sponsored">Paid Link</a>

<!-- User-generated untrusted link -->
<a href="https://example.com" rel="nofollow ugc">User Link</a>
```

## Security Rel Attributes

### noopener

**Prevents opened page from accessing window.opener.**

```html
<a href="https://external.com" target="_blank" rel="noopener">External Link</a>
```

**Always use with target="_blank"** to prevent:
- Tabnabbing attacks
- Performance issues
- Window.opener access

### noreferrer

**Prevents sending Referer header to linked page.**

```html
<a href="https://external.com" rel="noreferrer">Private Link</a>
```

**When to use:**
- Privacy-sensitive links
- When you don't want the destination to know where traffic came from
- Internal admin links

### External Links Best Practice

```html
<!-- ✅ CORRECT: External link with security attributes -->
<a href="https://external.com" target="_blank" rel="noopener noreferrer">
  External Site
</a>

<!-- ❌ WRONG: External link without protection -->
<a href="https://external.com" target="_blank">
  External Site
</a>
```

## Navigation Rel Attributes

### canonical

**Preferred URL for this page.**

```html
<link rel="canonical" href="https://example.com/page/">
```

See [canonical-urls.md](./canonical-urls.md) for details.

### alternate

**Alternative version of this page.**

```html
<!-- RSS feed -->
<link rel="alternate" type="application/rss+xml" href="/index.xml" title="RSS Feed">

<!-- Language alternatives -->
<link rel="alternate" hreflang="es" href="https://example.com/es/page/">

<!-- AMP version -->
<link rel="amphtml" href="https://example.com/amp/page/">
```

### prev / next

**Pagination navigation.**

```html
<link rel="prev" href="https://example.com/blog/page/2/">
<link rel="next" href="https://example.com/blog/page/4/">
```

**Hugo implementation:**

```go-html-template
{{ if .Paginator }}
  {{ if .Paginator.HasPrev }}
    <link rel="prev" href="{{ .Paginator.Prev.URL | absURL }}">
  {{ end }}
  {{ if .Paginator.HasNext }}
    <link rel="next" href="{{ .Paginator.Next.URL | absURL }}">
  {{ end }}
{{ end }}
```

### author

**Link to author information.**

```html
<link rel="author" href="/about/">
<a rel="author" href="/authors/jane-doe/">Jane Doe</a>
```

### license

**Link to content license.**

```html
<link rel="license" href="https://creativecommons.org/licenses/by/4.0/">
```

## Resource Hint Rel Attributes

### preconnect

**Establish early connections to origins.**

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://cdn.example.com" crossorigin>
```

**When to use:** Origins you'll definitely need soon (fonts, CDN, API).

### dns-prefetch

**Resolve DNS for origins early.**

```html
<link rel="dns-prefetch" href="https://analytics.example.com">
```

**When to use:** Origins you might need (analytics, third-party widgets).

### prefetch

**Prefetch resources for future navigation.**

```html
<link rel="prefetch" href="/next-page.html">
<link rel="prefetch" href="/css/page2.css" as="style">
```

**When to use:** Resources likely needed on next page.

### preload

**Preload resources needed for current page.**

```html
<link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/css/critical.css" as="style">
<link rel="preload" href="/images/hero.jpg" as="image">
```

**When to use:** Critical resources for current page (fonts, above-fold images).

### modulepreload

**Preload JavaScript modules.**

```html
<link rel="modulepreload" href="/js/app.mjs">
```

## Other Rel Attributes

### manifest

**Link to web app manifest (PWA).**

```html
<link rel="manifest" href="/site.webmanifest">
```

### icon / apple-touch-icon

**Favicon and touch icons.**

```html
<link rel="icon" href="/favicon.ico" sizes="32x32">
<link rel="icon" href="/icon.svg" type="image/svg+xml">
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
```

### me

**Link to author's identity (IndieWeb).**

```html
<a rel="me" href="https://twitter.com/janedoe">Twitter</a>
<a rel="me" href="https://github.com/janedoe">GitHub</a>
```

### search

**Link to site search.**

```html
<link rel="search" type="application/opensearchdescription+xml" href="/opensearch.xml" title="Site Search">
```

### pingback / webmention

**IndieWeb protocols.**

```html
<link rel="pingback" href="https://example.com/xmlrpc">
<link rel="webmention" href="https://example.com/webmention">
```

## Hugo Implementation

### External Link Render Hook

**Automatically add security attributes to external links:**

**layouts/_default/_markup/render-link.html:**

```go-html-template
{{- $url := .Destination -}}
{{- $isExternal := hasPrefix $url "http" -}}

{{- if $isExternal -}}
  <a href="{{ $url }}" target="_blank" rel="noopener noreferrer"
    {{- with .Title }} title="{{ . }}"{{ end -}}>
    {{- .Text -}}
  </a>
{{- else -}}
  <a href="{{ $url }}"
    {{- with .Title }} title="{{ . }}"{{ end -}}>
    {{- .Text -}}
  </a>
{{- end -}}
```

### Resource Hints Template

**layouts/partials/head/resource-hints.html:**

```go-html-template
{{/* Preconnect to critical origins */}}
{{ range .Site.Params.preconnect }}
  <link rel="preconnect" href="{{ . }}" crossorigin>
{{ end }}

{{/* DNS prefetch for non-critical origins */}}
{{ range .Site.Params.dns_prefetch }}
  <link rel="dns-prefetch" href="{{ . }}">
{{ end }}

{{/* Preload critical fonts */}}
{{ range .Site.Params.preload_fonts }}
  <link rel="preload" href="{{ . }}" as="font" type="font/woff2" crossorigin>
{{ end }}
```

### Complete Head with Rel Attributes

```go-html-template
<head>
  <meta charset="utf-8">

  {{/* Canonical */}}
  <link rel="canonical" href="{{ .Permalink }}">

  {{/* Pagination */}}
  {{ if .Paginator }}
    {{ if .Paginator.HasPrev }}
      <link rel="prev" href="{{ .Paginator.Prev.URL | absURL }}">
    {{ end }}
    {{ if .Paginator.HasNext }}
      <link rel="next" href="{{ .Paginator.Next.URL | absURL }}">
    {{ end }}
  {{ end }}

  {{/* Hreflang */}}
  {{ range .Translations }}
    <link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">
  {{ end }}

  {{/* Feeds */}}
  {{ range .AlternativeOutputFormats }}
    {{ printf `<link rel="%s" type="%s" href="%s" title="%s">` .Rel .MediaType.Type .Permalink (printf "%s - %s" $.Site.Title .Name) | safeHTML }}
  {{ end }}

  {{/* Favicon */}}
  <link rel="icon" href="/favicon.ico" sizes="32x32">
  <link rel="icon" href="/icon.svg" type="image/svg+xml">
  <link rel="apple-touch-icon" href="/apple-touch-icon.png">
  <link rel="manifest" href="/site.webmanifest">

  {{/* Resource hints */}}
  {{ partial "head/resource-hints.html" . }}

  {{/* License */}}
  {{ with .Site.Params.license_url }}
    <link rel="license" href="{{ . }}">
  {{ end }}
</head>
```

## Best Practices

### Security

**✅ DO:**
- Always use `rel="noopener"` with `target="_blank"`
- Add `rel="noreferrer"` for privacy-sensitive links
- Use render hooks to automate external link attributes

**❌ DON'T:**
- Open external links in new tabs without noopener
- Forget noreferrer for sensitive contexts
- Skip security attributes on user-generated links

### SEO

**✅ DO:**
- Use `rel="nofollow"` for untrusted links
- Use `rel="sponsored"` for paid links
- Use `rel="ugc"` for user-generated content
- Include `rel="canonical"` on every page
- Add `rel="alternate"` for feeds and translations

**❌ DON'T:**
- Add nofollow to all external links (only untrusted/paid)
- Forget canonical URLs
- Skip hreflang alternates for multilingual sites

### Performance

**✅ DO:**
- Preconnect to critical third-party origins (2-3 max)
- DNS prefetch for less critical origins
- Preload critical fonts and CSS
- Use prefetch for likely next pages

**❌ DON'T:**
- Preconnect to too many origins (diminishing returns after 3-5)
- Preload non-critical resources
- Prefetch resources that may never be needed

## Guidelines

### Essential

**Every page must have:**
- `rel="canonical"` (canonical URL)
- `rel="noopener"` on external `target="_blank"` links
- `rel="icon"` (favicon)
- `rel="alternate"` (RSS feed)

### Recommended

**For better SEO and security:**
- `rel="nofollow"` on untrusted links
- `rel="sponsored"` on paid links
- `rel="prev"` / `rel="next"` (pagination)
- `rel="alternate" hreflang` (multilingual)
- `rel="preconnect"` (critical origins)
- `rel="manifest"` (PWA)

### Advanced

**For maximum optimization:**
- Render hooks for automatic external link handling
- `rel="preload"` for critical resources
- `rel="prefetch"` for next-page resources
- `rel="me"` for IndieWeb identity
- `rel="webmention"` for IndieWeb
- `rel="license"` for content licensing

## Benefits

Security. Prevent tabnabbing and data leaks with noopener/noreferrer.

SEO Control. Manage link equity flow with nofollow/sponsored/ugc.

Performance. Resource hints improve page load speed.

Semantics. Rel attributes define clear relationships between resources.

Standards. HTML standard attributes supported by all browsers.

## Related

- [canonical-urls.md](./canonical-urls.md) - rel="canonical" detailed guide
- [hreflang-tags.md](./hreflang-tags.md) - rel="alternate" hreflang
- [link-prefetching-dns-prefetch.md](./link-prefetching-dns-prefetch.md) - Resource hints
- [meta-tags-comprehensive.md](./meta-tags-comprehensive.md) - All HTML meta tags
- [rss-feeds.md](./rss-feeds.md) - rel="alternate" for feeds
