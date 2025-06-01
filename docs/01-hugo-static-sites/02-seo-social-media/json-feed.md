# JSON Feed

JSON Feed 1.1 specification. JSON-based syndication. Modern feed format. Hugo JSON Feed template.

## Principle

Generate JSON Feed as a modern, developer-friendly alternative to RSS and Atom. Use native JSON format that's easy to parse and generate. Provide clean, well-structured content feeds.

## What is JSON Feed?

**JSON Feed:** JSON-based syndication format

**Purpose:** Modern alternative to RSS/Atom that uses JSON instead of XML

**Advantages:**
- Native JSON (no XML parsing)
- Simple, flat structure
- Easy to read and debug
- Easy to generate from any language
- Familiar to web developers

**Specification:** https://www.jsonfeed.org/version/1.1/

**Version:** 1.1 (current)

## JSON Feed vs RSS vs Atom

| Feature | JSON Feed | RSS 2.0 | Atom 1.0 |
|---------|-----------|---------|----------|
| Format | JSON | XML | XML |
| Parsing | Native JSON | XML parser | XML parser |
| Readability | High | Medium | Medium |
| Specification | jsonfeed.org | Informal | RFC 4287 |
| Reader support | Growing | Universal | Universal |
| Generation | Trivial | Moderate | Moderate |
| Debugging | Easy (JSON) | Harder (XML) | Harder (XML) |

**Recommendation:** Support all three formats for maximum compatibility.

## JSON Feed Structure

### Basic Feed

```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "Hugo Best Practices",
  "home_page_url": "https://example.com/",
  "feed_url": "https://example.com/feed.json",
  "description": "Learn to build fast static sites with Hugo",
  "language": "en-US",
  "authors": [
    {
      "name": "Hugo Team",
      "url": "https://example.com/about/"
    }
  ],
  "items": [
    {
      "id": "https://example.com/blog/hugo-performance-tips/",
      "url": "https://example.com/blog/hugo-performance-tips/",
      "title": "10 Hugo Performance Tips",
      "content_html": "<p>Full article content...</p>",
      "summary": "Learn 10 proven techniques to optimize Hugo site performance.",
      "date_published": "2026-02-13T10:00:00Z",
      "date_modified": "2026-02-14T09:15:00Z",
      "authors": [
        {
          "name": "Jane Doe"
        }
      ],
      "tags": ["Hugo", "Performance"]
    }
  ]
}
```

### Required Fields

**Feed level:**
- `version` - Must be `"https://jsonfeed.org/version/1.1"`
- `title` - Feed title
- `items` - Array of feed items

**Item level:**
- `id` - Unique identifier (URL recommended)

### Optional Fields

**Feed level:**
- `home_page_url` - Website URL
- `feed_url` - Self-referencing feed URL
- `description` - Feed description
- `user_comment` - Note for feed reader developers
- `next_url` - URL of next page (pagination)
- `icon` - Feed icon (512×512 minimum)
- `favicon` - Feed favicon
- `authors` - Array of author objects
- `language` - BCP 47 language tag
- `expired` - Boolean, true if feed is finished

**Item level:**
- `url` - Item URL
- `external_url` - URL of linked page (for link blogs)
- `title` - Item title
- `content_html` - Full HTML content
- `content_text` - Plain text content
- `summary` - Plain text summary
- `image` - Main image URL
- `banner_image` - Banner/hero image URL
- `date_published` - Publication date (RFC 3339)
- `date_modified` - Last modified date (RFC 3339)
- `authors` - Array of author objects
- `tags` - Array of tag strings
- `language` - Item-specific language
- `attachments` - Array of attachments (podcasts, etc.)

## Hugo Implementation

### Custom Output Format

**config.toml:**

```toml
[outputFormats]
  [outputFormats.JSONFEED]
    mediaType = "application/feed+json"
    baseName = "feed"
    isPlainText = true

[outputs]
  home = ["HTML", "RSS", "JSONFEED"]
  section = ["HTML", "RSS", "JSONFEED"]

[mediaTypes]
  [mediaTypes."application/feed+json"]
    suffixes = ["json"]
```

### JSON Feed Template

**layouts/index.jsonfeed.json:**

```go-html-template
{{- $pctx := . -}}
{{- if .IsHome -}}{{ $pctx = .Site }}{{- end -}}
{{- $pages := $pctx.RegularPages -}}
{{- $limit := .Site.Config.Services.RSS.Limit -}}
{{- if ge $limit 1 -}}
{{- $pages = $pages | first $limit -}}
{{- end -}}
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": {{ .Site.Title | jsonify }},
  "home_page_url": {{ .Site.BaseURL | jsonify }},
  "feed_url": {{ printf "%sfeed.json" .Site.BaseURL | jsonify }},
  {{ with .Site.Params.description }}
  "description": {{ . | jsonify }},
  {{ end }}
  "language": {{ .Site.Language.Lang | jsonify }},
  {{ with .Site.Params.author }}
  "authors": [
    {
      "name": {{ . | jsonify }}
      {{ with $.Site.Params.author_url }},
      "url": {{ . | jsonify }}
      {{ end }}
    }
  ],
  {{ end }}
  {{ with .Site.Params.og_image }}
  "icon": {{ . | absURL | jsonify }},
  {{ end }}
  "favicon": {{ "/favicon.ico" | absURL | jsonify }},
  "items": [
    {{- range $index, $page := $pages -}}
    {{- if $index }},{{ end }}
    {
      "id": {{ .Permalink | jsonify }},
      "url": {{ .Permalink | jsonify }},
      "title": {{ .Title | jsonify }},
      {{ with .Content }}
      "content_html": {{ . | jsonify }},
      {{ end }}
      {{ with .Description }}
      "summary": {{ . | jsonify }},
      {{ else }}
      {{ with .Summary }}
      "summary": {{ . | plainify | jsonify }},
      {{ end }}
      {{ end }}
      {{ with .Params.images }}
      "image": {{ index . 0 | absURL | jsonify }},
      {{ end }}
      {{ with .Params.banner_image }}
      "banner_image": {{ . | absURL | jsonify }},
      {{ end }}
      "date_published": {{ .Date.Format "2006-01-02T15:04:05Z07:00" | jsonify }},
      {{ if ne .Lastmod .Date }}
      "date_modified": {{ .Lastmod.Format "2006-01-02T15:04:05Z07:00" | jsonify }},
      {{ end }}
      {{ with .Params.author }}
      "authors": [
        {
          "name": {{ . | jsonify }}
        }
      ],
      {{ end }}
      {{ with .Params.tags }}
      "tags": {{ . | jsonify }}
      {{ else }}
      "tags": []
      {{ end }}
    }
    {{- end }}
  ]
}
```

### Section Feed Template

**layouts/_default/list.jsonfeed.json:**

```go-html-template
{{- $pages := .RegularPages -}}
{{- $limit := .Site.Config.Services.RSS.Limit -}}
{{- if ge $limit 1 -}}
{{- $pages = $pages | first $limit -}}
{{- end -}}
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": {{ printf "%s - %s" .Title .Site.Title | jsonify }},
  "home_page_url": {{ .Permalink | jsonify }},
  "feed_url": {{ printf "%sfeed.json" .Permalink | jsonify }},
  {{ with .Description }}
  "description": {{ . | jsonify }},
  {{ else }}
  "description": {{ printf "Recent %s on %s" .Title .Site.Title | jsonify }},
  {{ end }}
  "language": {{ .Site.Language.Lang | jsonify }},
  "items": [
    {{- range $index, $page := $pages -}}
    {{- if $index }},{{ end }}
    {
      "id": {{ .Permalink | jsonify }},
      "url": {{ .Permalink | jsonify }},
      "title": {{ .Title | jsonify }},
      {{ with .Content }}
      "content_html": {{ . | jsonify }},
      {{ end }}
      "summary": {{ with .Description }}{{ . | jsonify }}{{ else }}{{ .Summary | plainify | jsonify }}{{ end }},
      "date_published": {{ .Date.Format "2006-01-02T15:04:05Z07:00" | jsonify }},
      {{ with .Params.tags }}
      "tags": {{ . | jsonify }}
      {{ else }}
      "tags": []
      {{ end }}
    }
    {{- end }}
  ]
}
```

### Feed Discovery

**Add JSON Feed to HTML head:**

```go-html-template
{{/* JSON Feed discovery */}}
<link rel="alternate" type="application/feed+json" title="{{ .Site.Title }} - JSON Feed" href="{{ .Site.BaseURL }}feed.json">

{{/* RSS discovery */}}
<link rel="alternate" type="application/rss+xml" title="{{ .Site.Title }} - RSS" href="{{ .Site.BaseURL }}index.xml">

{{/* Atom discovery */}}
<link rel="alternate" type="application/atom+xml" title="{{ .Site.Title }} - Atom" href="{{ .Site.BaseURL }}atom.xml">
```

## Podcast Attachments

### Audio Attachment

```json
{
  "id": "https://example.com/podcast/episode-5/",
  "url": "https://example.com/podcast/episode-5/",
  "title": "Episode 5: Hugo Performance",
  "content_html": "<p>In this episode...</p>",
  "date_published": "2026-02-13T10:00:00Z",
  "attachments": [
    {
      "url": "https://cdn.example.com/episodes/ep005.mp3",
      "mime_type": "audio/mpeg",
      "title": "Episode 5 Audio",
      "size_in_bytes": 45000000,
      "duration_in_seconds": 2730
    }
  ]
}
```

### Hugo Template for Podcasts

```go-html-template
{{ with .Params.audio_url }}
"attachments": [
  {
    "url": {{ . | jsonify }},
    "mime_type": {{ $.Params.audio_type | default "audio/mpeg" | jsonify }},
    "title": {{ printf "%s Audio" $.Title | jsonify }},
    {{ with $.Params.audio_size }}
    "size_in_bytes": {{ . }},
    {{ end }}
    {{ with $.Params.duration_seconds }}
    "duration_in_seconds": {{ . }}
    {{ end }}
  }
],
{{ end }}
```

## Configuration

### Site Configuration

**config.toml:**

```toml
baseURL = "https://example.com"
title = "Hugo Best Practices"
languageCode = "en-us"

[params]
  description = "Learn to build fast static sites with Hugo"
  author = "Hugo Team"
  author_url = "https://example.com/about/"
  email = "hello@example.com"
  og_image = "/images/logo-512.png"

[outputFormats]
  [outputFormats.JSONFEED]
    mediaType = "application/feed+json"
    baseName = "feed"
    isPlainText = true

[mediaTypes]
  [mediaTypes."application/feed+json"]
    suffixes = ["json"]

[outputs]
  home = ["HTML", "RSS", "JSONFEED"]
  section = ["HTML", "RSS", "JSONFEED"]

[services]
  [services.rss]
    limit = 20
```

## Testing

### Validate JSON Feed

**JSON Feed Validator:**
- https://validator.jsonfeed.org/

**Manual validation:**

```bash
# Check feed exists and is valid JSON
curl -s https://example.com/feed.json | jq .

# Check version
curl -s https://example.com/feed.json | jq '.version'

# Count items
curl -s https://example.com/feed.json | jq '.items | length'

# Check item titles
curl -s https://example.com/feed.json | jq '.items[].title'
```

### Hugo Build Check

```bash
hugo
cat public/feed.json | jq .
cat public/blog/feed.json | jq .
```

### Feed Reader Testing

**JSON Feed supported readers:**
- NetNewsWire
- Feedbin
- Inoreader
- NewsBlur
- miniflux

## Common Issues

### Invalid JSON

**Problem:** Feed fails to parse

**Causes:**
- Trailing commas
- Unescaped characters in strings
- Missing quotes

**Solution:**

Always use Hugo's `| jsonify` filter:

```go-html-template
"title": {{ .Title | jsonify }},
```

This properly escapes special characters.

### Missing Version Field

**Problem:** Feed not recognized as JSON Feed

**Solution:**

Version must be exact string:

```json
"version": "https://jsonfeed.org/version/1.1"
```

### Empty Items Array

**Problem:** Feed has no items

**Solution:**

Check content exists:

```bash
hugo list all
```

Ensure pages are not all drafts.

## Best Practices

**✅ DO:**
- Include version field (required)
- Use `| jsonify` for all string values
- Provide both content_html and summary
- Include date_published (RFC 3339)
- Add feed_url (self-referencing)
- Validate JSON syntax
- Support alongside RSS/Atom

**❌ DON'T:**
- Manually construct JSON strings
- Skip escaping (use jsonify)
- Include only titles
- Forget version field
- Use invalid dates
- Break JSON syntax with trailing commas

## Guidelines

### Essential

**Minimum valid JSON Feed:**
- `version` (https://jsonfeed.org/version/1.1)
- `title`
- `items` array with at least `id` per item

### Recommended

**For better experience:**
- `home_page_url` and `feed_url`
- `description`
- `authors`
- Item `url`, `title`, `content_html`
- Item `date_published`
- Item `tags`
- Feed discovery in HTML head

### Advanced

**For maximum utility:**
- Podcast attachments
- Section-specific feeds
- All three formats (RSS + Atom + JSON)
- Feed pagination (next_url)
- Banner images

## Benefits

Developer Friendly. Native JSON is easy to parse and generate.

Modern Format. Designed for the modern web ecosystem.

Simple Structure. Flat, readable, debuggable.

Growing Support. Increasing feed reader support.

Easy Integration. Works with any JSON-capable tool or language.

## Related

- [rss-feeds.md](./rss-feeds.md) - RSS 2.0 feed format
- [atom-feeds.md](./atom-feeds.md) - Atom 1.0 feed format
- [meta-tags-comprehensive.md](./meta-tags-comprehensive.md) - Feed discovery links
