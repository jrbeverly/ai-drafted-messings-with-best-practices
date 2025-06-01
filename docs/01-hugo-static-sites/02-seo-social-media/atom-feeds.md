# Atom Feeds

Atom 1.0 syndication. RFC 4287. Content feeds. Hugo Atom template. Feed comparison.

## Principle

Generate Atom feeds as a standards-compliant alternative to RSS. Follow RFC 4287 specification. Provide richer metadata and better internationalization support. Use alongside or instead of RSS for content syndication.

## What is Atom?

**Atom:** XML-based syndication format (RFC 4287)

**Purpose:** Standardized content syndication with richer metadata than RSS

**Advantages over RSS:**
- IETF standard (RFC 4287) vs informal specification
- Better content typing (text, html, xhtml)
- Required unique IDs per entry
- Better internationalization (xml:lang)
- Consistent date format (ISO 8601/RFC 3339)
- Required self-referencing link

**Specification:** https://datatracker.ietf.org/doc/html/rfc4287

## Atom vs RSS

| Feature | Atom 1.0 | RSS 2.0 |
|---------|----------|---------|
| Standard | IETF RFC 4287 | Informal spec |
| Date format | RFC 3339 (ISO 8601) | RFC 822 |
| Content types | text, html, xhtml | text only |
| Unique IDs | Required | Optional (guid) |
| Self-link | Required | Optional |
| Internationalization | xml:lang support | Limited |
| Extension support | Built-in | Via namespaces |
| Reader support | Universal | Universal |

**Recommendation:** Support both for maximum compatibility.

## Atom Feed Structure

### Basic Atom Feed

```xml
<?xml version="1.0" encoding="utf-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">
  <title>Hugo Best Practices</title>
  <subtitle>Learn to build fast static sites with Hugo</subtitle>
  <link href="https://example.com/"/>
  <link href="https://example.com/atom.xml" rel="self"/>
  <id>https://example.com/</id>
  <updated>2026-02-13T10:00:00Z</updated>
  <author>
    <name>Hugo Team</name>
    <email>hello@example.com</email>
  </author>
  <generator uri="https://gohugo.io/" version="0.120.0">Hugo</generator>
  <rights>Copyright 2026 Hugo Best Practices</rights>

  <entry>
    <title>10 Hugo Performance Tips</title>
    <link href="https://example.com/blog/hugo-performance-tips/"/>
    <id>https://example.com/blog/hugo-performance-tips/</id>
    <published>2026-02-13T10:00:00Z</published>
    <updated>2026-02-14T09:15:00Z</updated>
    <author>
      <name>Jane Doe</name>
    </author>
    <summary type="html">Learn 10 proven techniques to optimize Hugo site performance.</summary>
    <content type="html">Full article HTML content here...</content>
    <category term="Hugo"/>
    <category term="Performance"/>
  </entry>
</feed>
```

### Required Elements

**Feed level:**
- `<title>` - Feed title
- `<id>` - Unique permanent feed identifier (URI)
- `<updated>` - Last time feed was modified (RFC 3339)
- `<link rel="self">` - Self-referencing URL

**Entry level:**
- `<title>` - Entry title
- `<id>` - Unique permanent entry identifier (URI)
- `<updated>` - Last time entry was modified

### Optional Elements

**Feed level:**
- `<subtitle>` - Feed description
- `<link>` - Website URL (without rel="self")
- `<author>` - Default author
- `<contributor>` - Additional contributors
- `<generator>` - Software that generated the feed
- `<icon>` - Feed icon (favicon)
- `<logo>` - Feed logo image
- `<rights>` - Copyright information
- `<category>` - Feed categories

**Entry level:**
- `<link>` - Entry URL
- `<author>` - Entry author
- `<published>` - Original publication date
- `<summary>` - Entry summary/excerpt
- `<content>` - Full entry content
- `<category>` - Entry categories
- `<contributor>` - Additional contributors
- `<rights>` - Entry-specific copyright

## Hugo Implementation

### Custom Output Format

**config.toml:**

```toml
[outputFormats]
  [outputFormats.ATOM]
    mediaType = "application/atom+xml"
    baseName = "atom"
    isPlainText = false

[outputs]
  home = ["HTML", "RSS", "ATOM"]
  section = ["HTML", "RSS", "ATOM"]
```

### Atom Feed Template

**layouts/_default/list.atom.xml:**

```xml
{{- $pctx := . -}}
{{- if .IsHome -}}{{ $pctx = .Site }}{{- end -}}
{{- $pages := slice -}}
{{- if or $.IsHome $.IsSection -}}
{{- $pages = $pctx.RegularPages -}}
{{- else -}}
{{- $pages = $pctx.Pages -}}
{{- end -}}
{{- $limit := .Site.Config.Services.RSS.Limit -}}
{{- if ge $limit 1 -}}
{{- $pages = $pages | first $limit -}}
{{- end -}}
{{- printf "<?xml version=\"1.0\" encoding=\"utf-8\"?>" | safeHTML }}
<feed xmlns="http://www.w3.org/2005/Atom" xml:lang="{{ .Site.Language.Lang }}">
  <title>{{ if eq .Title .Site.Title }}{{ .Site.Title }}{{ else }}{{ with .Title }}{{ . }} - {{ end }}{{ .Site.Title }}{{ end }}</title>
  {{ with .Site.Params.description }}
  <subtitle>{{ . }}</subtitle>
  {{ end }}
  <link href="{{ .Permalink }}" rel="alternate"/>
  {{ with .OutputFormats.Get "ATOM" }}
  <link href="{{ .Permalink }}" rel="self" type="application/atom+xml"/>
  {{ end }}
  <id>{{ .Permalink }}</id>
  {{ with $pages }}
  <updated>{{ (index . 0).Lastmod.Format "2006-01-02T15:04:05Z07:00" }}</updated>
  {{ else }}
  <updated>{{ now.Format "2006-01-02T15:04:05Z07:00" }}</updated>
  {{ end }}
  {{ with .Site.Params.author }}
  <author>
    <name>{{ . }}</name>
    {{ with $.Site.Params.email }}
    <email>{{ . }}</email>
    {{ end }}
    {{ with $.Site.Params.author_url }}
    <uri>{{ . }}</uri>
    {{ end }}
  </author>
  {{ end }}
  <generator uri="https://gohugo.io/" version="{{ hugo.Version }}">Hugo</generator>
  {{ with .Site.Params.og_image }}
  <logo>{{ . | absURL }}</logo>
  {{ end }}
  <icon>{{ "/favicon.ico" | absURL }}</icon>
  {{ with .Site.Params.copyright }}
  <rights>{{ . }}</rights>
  {{ end }}
  {{ range $pages }}
  <entry>
    <title>{{ .Title }}</title>
    <link href="{{ .Permalink }}" rel="alternate"/>
    <id>{{ .Permalink }}</id>
    <published>{{ .Date.Format "2006-01-02T15:04:05Z07:00" }}</published>
    <updated>{{ .Lastmod.Format "2006-01-02T15:04:05Z07:00" }}</updated>
    {{ with .Params.author }}
    <author>
      <name>{{ . }}</name>
    </author>
    {{ else }}
    {{ with $.Site.Params.author }}
    <author>
      <name>{{ . }}</name>
    </author>
    {{ end }}
    {{ end }}
    {{ range .Params.categories }}
    <category term="{{ . }}"/>
    {{ end }}
    {{ range .Params.tags }}
    <category term="{{ . }}"/>
    {{ end }}
    <summary type="html">{{ with .Description }}{{ . | html }}{{ else }}{{ .Summary | html }}{{ end }}</summary>
    {{ with .Content }}
    <content type="html">{{ . | html }}</content>
    {{ end }}
  </entry>
  {{ end }}
</feed>
```

### Home-Only Atom Template

**layouts/index.atom.xml:**

```xml
{{- $pages := .Site.RegularPages | first 20 -}}
{{- printf "<?xml version=\"1.0\" encoding=\"utf-8\"?>" | safeHTML }}
<feed xmlns="http://www.w3.org/2005/Atom">
  <title>{{ .Site.Title }}</title>
  <subtitle>{{ .Site.Params.description }}</subtitle>
  <link href="{{ .Site.BaseURL }}" rel="alternate"/>
  <link href="{{ .Site.BaseURL }}atom.xml" rel="self" type="application/atom+xml"/>
  <id>{{ .Site.BaseURL }}</id>
  <updated>{{ (index $pages 0).Lastmod.Format "2006-01-02T15:04:05Z07:00" }}</updated>
  <generator uri="https://gohugo.io/">Hugo</generator>
  {{ range $pages }}
  <entry>
    <title>{{ .Title }}</title>
    <link href="{{ .Permalink }}" rel="alternate"/>
    <id>{{ .Permalink }}</id>
    <published>{{ .Date.Format "2006-01-02T15:04:05Z07:00" }}</published>
    <updated>{{ .Lastmod.Format "2006-01-02T15:04:05Z07:00" }}</updated>
    <summary type="html">{{ .Summary | html }}</summary>
  </entry>
  {{ end }}
</feed>
```

### Feed Discovery

**Add Atom feed to HTML head:**

```go-html-template
{{/* Atom feed discovery */}}
<link rel="alternate" type="application/atom+xml" title="{{ .Site.Title }} - Atom Feed" href="{{ .Site.BaseURL }}atom.xml">

{{/* RSS feed discovery */}}
<link rel="alternate" type="application/rss+xml" title="{{ .Site.Title }} - RSS Feed" href="{{ .Site.BaseURL }}index.xml">
```

**Or use Hugo's automatic approach:**

```go-html-template
{{ range .AlternativeOutputFormats }}
  {{ printf `<link rel="%s" type="%s" href="%s" title="%s">` .Rel .MediaType.Type .Permalink (printf "%s - %s" $.Site.Title .Name) | safeHTML }}
{{ end }}
```

## Content Types

### Text Content

```xml
<summary type="text">Plain text summary without HTML.</summary>
```

### HTML Content

```xml
<summary type="html">&lt;p&gt;HTML summary with &lt;strong&gt;formatting&lt;/strong&gt;.&lt;/p&gt;</summary>
<content type="html">&lt;p&gt;Full HTML content...&lt;/p&gt;</content>
```

### XHTML Content

```xml
<content type="xhtml">
  <div xmlns="http://www.w3.org/1999/xhtml">
    <p>XHTML content with <strong>formatting</strong>.</p>
  </div>
</content>
```

## Configuration

### Site Configuration

**config.toml:**

```toml
baseURL = "https://example.com"
title = "Hugo Best Practices"
languageCode = "en-us"
copyright = "Copyright 2026 Hugo Best Practices"

[params]
  description = "Learn to build fast static sites with Hugo"
  author = "Hugo Team"
  email = "hello@example.com"
  author_url = "https://example.com/about/"
  og_image = "/images/logo.png"

[outputFormats]
  [outputFormats.ATOM]
    mediaType = "application/atom+xml"
    baseName = "atom"
    isPlainText = false

[outputs]
  home = ["HTML", "RSS", "ATOM"]
  section = ["HTML", "RSS", "ATOM"]

[services]
  [services.rss]
    limit = 20
```

### Media Type Registration

**If Hugo doesn't recognize atom media type:**

```toml
[mediaTypes]
  [mediaTypes."application/atom+xml"]
    suffixes = ["xml"]
```

## Testing

### Validate Atom Feed

**W3C Feed Validation:**
- https://validator.w3.org/feed/

**Manual check:**

```bash
# Check feed exists
curl -s https://example.com/atom.xml | head -20

# Validate XML syntax
curl -s https://example.com/atom.xml | xmllint --noout -

# Check required elements
curl -s https://example.com/atom.xml | grep '<id>'
curl -s https://example.com/atom.xml | grep '<updated>'
curl -s https://example.com/atom.xml | grep 'rel="self"'
```

### Feed Reader Testing

**Test in feed readers:**
- Feedly
- Inoreader
- NetNewsWire
- Thunderbird
- miniflux

### Hugo Build Check

```bash
hugo
cat public/atom.xml | head -30
cat public/blog/atom.xml | head -30
```

## Best Practices

### Content

**✅ DO:**
- Include full content in `<content>` element
- Provide meaningful summary
- Use proper content types (text/html/xhtml)
- Include publication and updated dates
- Add categories and tags
- Provide author information
- Use unique permanent IDs

**❌ DON'T:**
- Include only titles
- Skip content type attribute
- Use invalid date format
- Reuse IDs across entries
- Change entry IDs after publication
- Skip self-referencing link

### Technical

**✅ DO:**
- Include required elements (title, id, updated, self-link)
- Use RFC 3339 date format (ISO 8601)
- Validate with W3C validator
- Include xml:lang for internationalization
- Use absolute URLs
- Escape HTML in content

**❌ DON'T:**
- Break XML syntax
- Use relative URLs
- Skip required elements
- Change feed URL without redirect
- Include malformed dates

## Guidelines

### Essential

**Minimum valid Atom feed:**
- `<feed>` with xmlns attribute
- `<title>` - Feed title
- `<id>` - Unique feed URI
- `<updated>` - Feed last modified (RFC 3339)
- `<link rel="self">` - Self-referencing URL
- At least one `<entry>` with title, id, updated

### Recommended

**For better experience:**
- `<subtitle>` - Feed description
- `<author>` - Default author
- `<generator>` - Hugo generator info
- Entry `<content>` - Full content
- Entry `<summary>` - Excerpt
- Entry `<published>` - Original date
- Entry `<category>` - Tags/categories
- `<logo>` and `<icon>` - Branding

### Advanced

**For maximum utility:**
- Both RSS and Atom feeds
- Section-specific feeds
- Full XHTML content
- Contributor metadata
- Feed pagination (RFC 5005)
- Feed categories

## Benefits

Standards Compliant. IETF RFC 4287 standard.

Rich Metadata. Better content typing and internationalization.

Unique IDs. Required unique identifiers prevent duplicates.

Better Dates. ISO 8601/RFC 3339 format is unambiguous.

Universal Support. Works in all major feed readers.

## Related

- [rss-feeds.md](./rss-feeds.md) - RSS 2.0 feed format
- [json-feed.md](./json-feed.md) - JSON Feed format
- [dublin-core-metadata.md](./dublin-core-metadata.md) - Dublin Core metadata
- [meta-tags-comprehensive.md](./meta-tags-comprehensive.md) - Feed discovery links
