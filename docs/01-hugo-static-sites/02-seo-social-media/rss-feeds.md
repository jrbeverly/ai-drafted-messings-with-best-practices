# RSS Feeds

RSS 2.0 feeds. Content syndication. Hugo RSS template. Feed customization. Podcast feeds.

## Principle

Generate RSS feeds to enable content syndication and subscription. Use Hugo's built-in RSS support for automatic feed generation. Customize feed content for optimal reader experience. Follow RSS 2.0 specification.

## What is RSS?

**RSS:** Really Simple Syndication (version 2.0)

**Purpose:** XML-based format for distributing content updates

**Use cases:**
- Blog subscriptions (Feedly, Inoreader)
- Podcast distribution (Apple Podcasts, Spotify)
- Content aggregation
- News readers
- Automated workflows (IFTTT, Zapier)

**Specification:** https://www.rssboard.org/rss-specification

## Hugo Built-in RSS

Hugo generates RSS feeds automatically.

**Default output:** `/index.xml`

**Feeds generated:**
- `/index.xml` - All site content
- `/blog/index.xml` - Blog section feed
- `/tags/hugo/index.xml` - Tag-specific feed
- `/categories/tutorial/index.xml` - Category feed

### Default Configuration

**config.toml:**

```toml
[outputs]
  home = ["HTML", "RSS"]
  section = ["HTML", "RSS"]
  taxonomy = ["HTML", "RSS"]
  term = ["HTML", "RSS"]
```

### Default RSS Template

Hugo uses a built-in RSS template. Override with `layouts/_default/rss.xml`.

**Default output example:**

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>Hugo Best Practices</title>
    <link>https://example.com/</link>
    <description>Recent content on Hugo Best Practices</description>
    <generator>Hugo</generator>
    <language>en-us</language>
    <lastBuildDate>Thu, 13 Feb 2026 10:00:00 +0000</lastBuildDate>
    <atom:link href="https://example.com/index.xml" rel="self" type="application/rss+xml"/>
    <item>
      <title>10 Hugo Performance Tips</title>
      <link>https://example.com/blog/hugo-performance-tips/</link>
      <pubDate>Thu, 13 Feb 2026 10:00:00 +0000</pubDate>
      <guid>https://example.com/blog/hugo-performance-tips/</guid>
      <description>Learn 10 proven techniques to optimize Hugo site performance.</description>
    </item>
  </channel>
</rss>
```

## Custom RSS Template

### Full Content Feed

**layouts/_default/rss.xml:**

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
{{- printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<rss version="2.0"
  xmlns:atom="http://www.w3.org/2005/Atom"
  xmlns:content="http://purl.org/rss/1.0/modules/content/"
  xmlns:dc="http://purl.org/dc/elements/1.1/"
  xmlns:media="http://search.yahoo.com/mrss/">
  <channel>
    <title>{{ if eq .Title .Site.Title }}{{ .Site.Title }}{{ else }}{{ with .Title }}{{ . }} - {{ end }}{{ .Site.Title }}{{ end }}</title>
    <link>{{ .Permalink }}</link>
    <description>{{ with .Site.Params.description }}{{ . }}{{ else }}Recent content on {{ .Site.Title }}{{ end }}</description>
    <generator>Hugo -- gohugo.io</generator>
    <language>{{ .Site.Language.Lang }}</language>
    {{ with .Site.Params.author }}
    <managingEditor>{{ $.Site.Params.email }} ({{ . }})</managingEditor>
    <webMaster>{{ $.Site.Params.email }} ({{ . }})</webMaster>
    {{ end }}
    {{ with .Site.Params.copyright }}
    <copyright>{{ . }}</copyright>
    {{ end }}
    <lastBuildDate>{{ .Date.Format "Mon, 02 Jan 2006 15:04:05 -0700" | safeHTML }}</lastBuildDate>
    {{ with .OutputFormats.Get "RSS" }}
      {{ printf "<atom:link href=%q rel=\"self\" type=%q />" .Permalink .MediaType | safeHTML }}
    {{ end }}
    {{ with .Site.Params.og_image }}
    <image>
      <url>{{ . | absURL }}</url>
      <title>{{ $.Site.Title }}</title>
      <link>{{ $.Site.BaseURL }}</link>
    </image>
    {{ end }}
    {{ range $pages }}
    <item>
      <title>{{ .Title }}</title>
      <link>{{ .Permalink }}</link>
      <pubDate>{{ .Date.Format "Mon, 02 Jan 2006 15:04:05 -0700" | safeHTML }}</pubDate>
      {{ with .Params.author }}
      <dc:creator>{{ . }}</dc:creator>
      {{ end }}
      {{ range .Params.categories }}
      <category>{{ . }}</category>
      {{ end }}
      {{ range .Params.tags }}
      <category>{{ . }}</category>
      {{ end }}
      <guid>{{ .Permalink }}</guid>
      <description>{{ with .Description }}{{ . | html }}{{ else }}{{ .Summary | html }}{{ end }}</description>
      {{ with .Content }}
      <content:encoded>{{ . | html }}</content:encoded>
      {{ end }}
      {{ with .Params.images }}
      {{ range first 1 . }}
      <media:content url="{{ . | absURL }}" medium="image"/>
      {{ end }}
      {{ end }}
    </item>
    {{ end }}
  </channel>
</rss>
```

### Summary-Only Feed

**Only include excerpt, not full content:**

```xml
<item>
  <title>{{ .Title }}</title>
  <link>{{ .Permalink }}</link>
  <pubDate>{{ .Date.Format "Mon, 02 Jan 2006 15:04:05 -0700" | safeHTML }}</pubDate>
  <guid>{{ .Permalink }}</guid>
  <description>{{ with .Description }}{{ . | html }}{{ else }}{{ .Summary | html }}{{ end }}</description>
  {{/* No content:encoded - summary only */}}
</item>
```

### Limit Number of Items

**config.toml:**

```toml
[services]
  [services.rss]
    limit = 20  # Maximum items in feed
```

**Or in template:**

```go-html-template
{{ $pages := $pctx.RegularPages | first 20 }}
```

## Feed Configuration

### Site Configuration

**config.toml:**

```toml
baseURL = "https://example.com"
title = "Hugo Best Practices"
languageCode = "en-us"
copyright = "Copyright 2026 Hugo Best Practices. All rights reserved."

[params]
  description = "Learn to build fast static sites with Hugo"
  author = "Hugo Team"
  email = "hello@example.com"
  og_image = "/images/logo.png"

[services]
  [services.rss]
    limit = 20

[outputs]
  home = ["HTML", "RSS"]
  section = ["HTML", "RSS"]
  taxonomy = ["HTML", "RSS"]
  term = ["HTML", "RSS"]
```

### Section-Specific Feeds

**Disable RSS for specific sections:**

```toml
# Only blog section gets RSS
[outputs]
  home = ["HTML", "RSS"]
  section = ["HTML", "RSS"]

# In content/docs/_index.md (disable for docs)
# outputs: ["HTML"]
```

**Or in section front matter:**

```yaml
# content/docs/_index.md
---
title: "Documentation"
outputs:
  - HTML
  # No RSS for docs section
---
```

### Feed Discovery

**Add feed links to HTML head:**

**layouts/partials/head/feeds.html:**

```go-html-template
{{/* RSS feed discovery */}}
{{ range .AlternativeOutputFormats }}
  {{ printf `<link rel="%s" type="%s" href="%s" title="%s">` .Rel .MediaType.Type .Permalink (printf "%s - %s" $.Site.Title .Name) | safeHTML }}
{{ end }}
```

**Output:**

```html
<link rel="alternate" type="application/rss+xml" href="https://example.com/index.xml" title="Hugo Best Practices - RSS">
```

### Custom Feed URL

**Change feed filename:**

```toml
[outputFormats]
  [outputFormats.RSS]
    baseName = "feed"  # /feed.xml instead of /index.xml
```

**Or:**

```toml
[outputFormats]
  [outputFormats.RSS]
    baseName = "rss"   # /rss.xml
```

## Podcast RSS Feed

### iTunes/Apple Podcasts Feed

**layouts/section/podcast-rss.xml:**

```xml
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<rss version="2.0"
  xmlns:atom="http://www.w3.org/2005/Atom"
  xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd"
  xmlns:content="http://purl.org/rss/1.0/modules/content/">
  <channel>
    <title>{{ .Site.Params.podcast_title }}</title>
    <link>{{ .Permalink }}</link>
    <description>{{ .Site.Params.podcast_description }}</description>
    <language>{{ .Site.Language.Lang }}</language>
    <lastBuildDate>{{ .Date.Format "Mon, 02 Jan 2006 15:04:05 -0700" | safeHTML }}</lastBuildDate>
    {{ with .OutputFormats.Get "RSS" }}
      {{ printf "<atom:link href=%q rel=\"self\" type=%q />" .Permalink .MediaType | safeHTML }}
    {{ end }}

    {{/* iTunes-specific tags */}}
    <itunes:author>{{ .Site.Params.podcast_author }}</itunes:author>
    <itunes:summary>{{ .Site.Params.podcast_description }}</itunes:summary>
    <itunes:owner>
      <itunes:name>{{ .Site.Params.podcast_author }}</itunes:name>
      <itunes:email>{{ .Site.Params.podcast_email }}</itunes:email>
    </itunes:owner>
    <itunes:image href="{{ .Site.Params.podcast_image | absURL }}"/>
    <itunes:category text="{{ .Site.Params.podcast_category }}">
      {{ with .Site.Params.podcast_subcategory }}
      <itunes:category text="{{ . }}"/>
      {{ end }}
    </itunes:category>
    <itunes:explicit>{{ .Site.Params.podcast_explicit | default "false" }}</itunes:explicit>

    <image>
      <url>{{ .Site.Params.podcast_image | absURL }}</url>
      <title>{{ .Site.Params.podcast_title }}</title>
      <link>{{ .Permalink }}</link>
    </image>

    {{ range .RegularPages }}
    <item>
      <title>{{ .Title }}</title>
      <link>{{ .Permalink }}</link>
      <pubDate>{{ .Date.Format "Mon, 02 Jan 2006 15:04:05 -0700" | safeHTML }}</pubDate>
      <guid>{{ .Permalink }}</guid>
      <description>{{ .Description | html }}</description>
      {{ with .Content }}
      <content:encoded>{{ . | html }}</content:encoded>
      {{ end }}

      {{/* Enclosure (audio file) */}}
      {{ with .Params.audio_url }}
      <enclosure url="{{ . }}" length="{{ $.Params.audio_size | default 0 }}" type="{{ $.Params.audio_type | default "audio/mpeg" }}"/>
      {{ end }}

      {{/* iTunes episode tags */}}
      <itunes:duration>{{ .Params.duration }}</itunes:duration>
      <itunes:author>{{ .Params.author | default $.Site.Params.podcast_author }}</itunes:author>
      <itunes:summary>{{ .Description }}</itunes:summary>
      {{ with .Params.episode_number }}
      <itunes:episode>{{ . }}</itunes:episode>
      {{ end }}
      {{ with .Params.season }}
      <itunes:season>{{ . }}</itunes:season>
      {{ end }}
      <itunes:episodeType>{{ .Params.episode_type | default "full" }}</itunes:episodeType>
      {{ with .Params.episode_image }}
      <itunes:image href="{{ . | absURL }}"/>
      {{ end }}
    </item>
    {{ end }}
  </channel>
</rss>
```

**Podcast episode front matter:**

```yaml
---
title: "Episode 5: Hugo Performance Optimization"
description: "Tips and techniques for making Hugo sites faster"
date: 2026-02-13T10:00:00Z
audio_url: https://cdn.example.com/episodes/ep005.mp3
audio_size: 45000000
audio_type: audio/mpeg
duration: "45:30"
episode_number: 5
season: 1
episode_type: full
episode_image: /images/podcast/ep005.jpg
---
```

## Testing RSS Feeds

### Validate Feed

**W3C Feed Validation Service:**
- https://validator.w3.org/feed/

**How to use:**
1. Enter feed URL
2. Click "Check"
3. Fix errors and warnings

### Manual Testing

```bash
# Check feed exists
curl -s https://example.com/index.xml | head -20

# Validate XML syntax
curl -s https://example.com/index.xml | xmllint --noout -

# Count items
curl -s https://example.com/index.xml | grep -c '<item>'

# Check specific elements
curl -s https://example.com/index.xml | grep '<title>'
```

### Hugo Build Testing

```bash
hugo
cat public/index.xml | head -30
cat public/blog/index.xml | head -30
```

### Feed Reader Testing

**Test in actual feed readers:**
- Feedly (https://feedly.com)
- Inoreader (https://www.inoreader.com)
- NetNewsWire (macOS/iOS)
- Thunderbird (desktop)

## Common Issues

### Feed Not Generating

**Problem:** `/index.xml` returns 404

**Solution:**

```toml
# Ensure RSS is in outputs
[outputs]
  home = ["HTML", "RSS"]
```

### Empty Feed

**Problem:** Feed has no items

**Causes:**
- No published content
- All content is draft
- Wrong section configuration

**Solution:**

```bash
hugo list all  # Check published pages
hugo list drafts  # Check drafts
```

### Wrong Date Format

**Problem:** Dates don't parse in feed readers

**Solution:**

Use RFC 822 format:

```go-html-template
{{ .Date.Format "Mon, 02 Jan 2006 15:04:05 -0700" | safeHTML }}
```

### Missing Full Content

**Problem:** Feed only shows summary

**Solution:**

Include `content:encoded`:

```xml
<content:encoded>{{ .Content | html }}</content:encoded>
```

### Special Characters in XML

**Problem:** Invalid XML due to special characters

**Solution:**

Use `| html` to escape:

```go-html-template
<description>{{ .Description | html }}</description>
```

## Best Practices

### Content

**✅ DO:**
- Include full content (or meaningful summary)
- Use descriptive titles
- Include publication dates
- Add categories/tags
- Limit feed to 20-50 items
- Include images where relevant
- Provide author information

**❌ DON'T:**
- Include only titles (useless feed)
- Use duplicate content across feeds
- Include drafts or private content
- Forget date formatting (RFC 822)
- Skip XML escaping

### Technical

**✅ DO:**
- Validate with W3C validator
- Include atom:link self-reference
- Use proper XML namespaces
- Include feed discovery in HTML head
- Set appropriate MIME type (application/rss+xml)
- Use HTTPS for feed URL

**❌ DON'T:**
- Break XML syntax
- Forget self-referencing atom:link
- Use relative URLs in feed
- Skip validation
- Change feed URL (break subscriptions)

### Discovery

**✅ DO:**
- Add `<link rel="alternate">` in HTML head
- Include feed URL in footer
- Mention RSS in about page
- Submit to feed directories
- Include in robots.txt (optional)

**❌ DON'T:**
- Hide feed from users
- Forget feed discovery link
- Change feed URL without redirect

## Guidelines

### Essential

**Minimum RSS feed:**
- Valid RSS 2.0 XML
- Channel title, link, description
- Item title, link, pubDate, guid
- Feed discovery in HTML head
- atom:link self-reference

### Recommended

**For better experience:**
- Full content (content:encoded)
- Author information (dc:creator)
- Categories/tags
- Images (media:content)
- Limit to 20 items
- Description/summary fallback

### Advanced

**For maximum utility:**
- Podcast feeds (iTunes tags)
- Section-specific feeds
- Custom feed URLs
- Multiple feed formats (RSS + Atom + JSON)
- Feed analytics

## Benefits

Content Distribution. Readers subscribe and get updates automatically.

SEO. Feed aggregators and crawlers discover new content.

Retention. Subscribers return to your site regularly.

Syndication. Content shared across platforms and aggregators.

Automation. Triggers workflows (IFTTT, Zapier, email newsletters).

## Related

- [atom-feeds.md](./atom-feeds.md) - Atom feed format
- [json-feed.md](./json-feed.md) - JSON Feed format
- [sitemaps-robots-txt.md](./sitemaps-robots-txt.md) - Include feed in robots.txt
- [meta-tags-comprehensive.md](./meta-tags-comprehensive.md) - Feed discovery link tags
- [dublin-core-metadata.md](./dublin-core-metadata.md) - Dublin Core in RSS feeds
