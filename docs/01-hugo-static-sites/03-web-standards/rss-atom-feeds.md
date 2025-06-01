# RSS and Atom Feeds

Content syndication feeds. RSS 2.0, Atom 1.0. Subscribe to updates. Feed readers. Podcast feeds.

## Principle

Enable users to subscribe to your content updates. Support feed readers and aggregators. Syndicate content automatically. Allow programmatic content consumption.

## What are RSS and Atom Feeds?

**RSS:** Really Simple Syndication - XML format for content syndication

**Atom:** Modern alternative to RSS with stricter specification

**Purpose:**
- Subscribe to blog/news updates
- Content syndication
- Podcast distribution
- Automated content consumption

**Feed Readers:** Feedly, The Old Reader, NewsBlur, NetNewsWire

**Format:** XML file (typically `/feed.xml`, `/rss.xml`, or `/index.xml`)

## RSS 2.0 Feed

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>Example Blog</title>
    <link>https://example.com/</link>
    <description>Latest articles about web development</description>
    <language>en-us</language>
    <lastBuildDate>Wed, 13 Feb 2026 10:00:00 +0000</lastBuildDate>
    <atom:link href="https://example.com/feed.xml" rel="self" type="application/rss+xml"/>

    <item>
      <title>Complete Guide to RSS Feeds</title>
      <link>https://example.com/blog/rss-guide</link>
      <description>Learn how to implement RSS feeds for content syndication.</description>
      <pubDate>Wed, 13 Feb 2026 10:00:00 +0000</pubDate>
      <guid isPermaLink="true">https://example.com/blog/rss-guide</guid>
      <author>jane@example.com (Jane Doe)</author>
      <category>Web Development</category>
      <category>RSS</category>
    </item>

    <item>
      <title>Second Article Title</title>
      <link>https://example.com/blog/second-article</link>
      <description>Article description here.</description>
      <pubDate>Mon, 11 Feb 2026 15:30:00 +0000</pubDate>
      <guid isPermaLink="true">https://example.com/blog/second-article</guid>
    </item>
  </channel>
</rss>
```

### RSS Channel Elements

**Required:**
- `<title>`: Feed title
- `<link>`: Website URL
- `<description>`: Feed description

**Recommended:**
- `<language>`: Language code (en-us, es, fr, etc.)
- `<lastBuildDate>`: Last update date
- `<atom:link rel="self">`: Feed URL (best practice)

**Optional:**
- `<copyright>`: Copyright notice
- `<managingEditor>`: Editor email
- `<webMaster>`: Webmaster email
- `<category>`: Feed category
- `<image>`: Feed logo

### RSS Item Elements

**Required:**
- `<title>` OR `<description>` (at least one)

**Recommended:**
- `<title>`: Article title
- `<link>`: Article URL
- `<description>`: Article summary or full content
- `<pubDate>`: Publication date
- `<guid>`: Unique identifier (usually permalink)

**Optional:**
- `<author>`: Author email
- `<category>`: Article categories/tags
- `<comments>`: Comments URL
- `<enclosure>`: Media file (podcasts)

## Atom 1.0 Feed

```xml
<?xml version="1.0" encoding="UTF-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">
  <title>Example Blog</title>
  <link href="https://example.com/"/>
  <link href="https://example.com/atom.xml" rel="self" type="application/atom+xml"/>
  <updated>2026-02-13T10:00:00Z</updated>
  <id>https://example.com/</id>
  <subtitle>Latest articles about web development</subtitle>
  <author>
    <name>Jane Doe</name>
    <email>jane@example.com</email>
    <uri>https://example.com/authors/jane</uri>
  </author>

  <entry>
    <title>Complete Guide to Atom Feeds</title>
    <link href="https://example.com/blog/atom-guide"/>
    <id>https://example.com/blog/atom-guide</id>
    <updated>2026-02-13T10:00:00Z</updated>
    <published>2026-02-13T10:00:00Z</published>
    <summary>Learn how to implement Atom feeds for content syndication.</summary>
    <content type="html">
      <![CDATA[
        <p>Full article HTML content here...</p>
      ]]>
    </content>
    <author>
      <name>Jane Doe</name>
      <email>jane@example.com</email>
    </author>
    <category term="Web Development"/>
    <category term="Atom"/>
  </entry>

  <entry>
    <title>Second Article</title>
    <link href="https://example.com/blog/second-article"/>
    <id>https://example.com/blog/second-article</id>
    <updated>2026-02-11T15:30:00Z</updated>
    <summary>Second article summary.</summary>
  </entry>
</feed>
```

### Atom Feed Elements

**Required:**
- `<title>`: Feed title
- `<link href="..." rel="self">`: Feed URL
- `<updated>`: Last update timestamp
- `<id>`: Unique feed identifier

**Recommended:**
- `<author>`: Feed author
- `<subtitle>`: Feed description

### Atom Entry Elements

**Required:**
- `<title>`: Entry title
- `<link>`: Entry URL
- `<id>`: Unique identifier
- `<updated>`: Last update timestamp

**Recommended:**
- `<summary>`: Entry summary
- `<content>`: Full entry content
- `<published>`: Publication date
- `<author>`: Entry author
- `<category>`: Entry categories

## RSS vs Atom

| Feature | RSS 2.0 | Atom 1.0 |
|---------|---------|----------|
| Specification | Loose | Strict |
| Namespace | Optional | Required |
| Dates | RFC 822 | ISO 8601 (RFC 3339) |
| Content | `<description>` | `<summary>` + `<content>` |
| Author | Email format | Structured |
| Self-link | Via Atom namespace | Native |
| Adoption | More common | More modern |

**Recommendation:** Use RSS 2.0 (more widely supported by feed readers)

## Hugo RSS Implementation

### Hugo Default RSS

Hugo automatically generates RSS at `/index.xml`

**Default URL:** `https://example.com/index.xml`

**Per-section RSS:** `https://example.com/blog/index.xml`

### Hugo RSS Configuration

```toml
# config.toml

[outputs]
home = ["HTML", "RSS"]
section = ["HTML", "RSS"]

[params]
description = "Blog about web development"
author = "Jane Doe"

[author]
name = "Jane Doe"
email = "jane@example.com"
```

### Custom Hugo RSS Template

```go-html-template
{{/* layouts/_default/rss.xml */}}
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>{{ if eq .Title .Site.Title }}{{ .Site.Title }}{{ else }}{{ with .Title }}{{ . }} on {{ end }}{{ .Site.Title }}{{ end }}</title>
    <link>{{ .Permalink }}</link>
    <description>{{ with .Description }}{{ . }}{{ else }}{{ .Site.Params.description }}{{ end }}</description>
    <language>{{ .Site.LanguageCode | default "en-us" }}</language>
    {{ with .Site.Author.email }}<managingEditor>{{ . }} ({{ $.Site.Author.name }})</managingEditor>{{ end }}
    {{ with .Site.Author.email }}<webMaster>{{ . }} ({{ $.Site.Author.name }})</webMaster>{{ end }}
    {{ with .Site.Copyright }}<copyright>{{ . }}</copyright>{{ end }}
    <lastBuildDate>{{ .Date.Format "Mon, 02 Jan 2006 15:04:05 -0700" }}</lastBuildDate>
    <atom:link href="{{ .Permalink }}" rel="self" type="application/rss+xml" />

    {{ range first 20 .Pages }}
    <item>
      <title>{{ .Title }}</title>
      <link>{{ .Permalink }}</link>
      <pubDate>{{ .Date.Format "Mon, 02 Jan 2006 15:04:05 -0700" }}</pubDate>
      {{ with .Site.Author.email }}<author>{{ . }} ({{ $.Site.Author.name }})</author>{{ end }}
      <guid isPermaLink="true">{{ .Permalink }}</guid>
      <description>{{ with .Description }}{{ . }}{{ else }}{{ .Summary | html }}{{ end }}</description>

      {{ range .Params.categories }}
      <category>{{ . }}</category>
      {{ end }}

      {{ range .Params.tags }}
      <category>{{ . }}</category>
      {{ end }}
    </item>
    {{ end }}
  </channel>
</rss>
```

### Full Content in RSS

```go-html-template
{{/* Include full HTML content instead of summary */}}
<item>
  <title>{{ .Title }}</title>
  <link>{{ .Permalink }}</link>
  <description>
    <![CDATA[{{ .Content }}]]>
  </description>
  <pubDate>{{ .Date.Format "Mon, 02 Jan 2006 15:04:05 -0700" }}</pubDate>
  <guid isPermaLink="true">{{ .Permalink }}</guid>
</item>
```

**Trade-off:** Full content vs summary (full content allows reading in feed reader, summary drives traffic to site)

## Feed Discovery (HTML Link Tag)

```html
<!-- Link to RSS feed in HTML <head> -->
<link rel="alternate" type="application/rss+xml" title="Blog RSS Feed" href="https://example.com/index.xml">

<!-- Atom feed -->
<link rel="alternate" type="application/atom+xml" title="Blog Atom Feed" href="https://example.com/atom.xml">
```

**Hugo template:**

```go-html-template
{{/* layouts/partials/head.html */}}

{{ with .OutputFormats.Get "RSS" }}
<link rel="alternate" type="application/rss+xml" title="{{ $.Site.Title }} RSS Feed" href="{{ .Permalink }}">
{{ end }}
```

## Podcast RSS Feed

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0"
     xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd"
     xmlns:content="http://purl.org/rss/1.0/modules/content/">
  <channel>
    <title>Example Podcast</title>
    <link>https://example.com/podcast</link>
    <description>Weekly podcast about web development</description>
    <language>en-us</language>
    <itunes:author>Jane Doe</itunes:author>
    <itunes:category text="Technology"/>
    <itunes:image href="https://example.com/podcast-cover.jpg"/>
    <itunes:explicit>no</itunes:explicit>

    <item>
      <title>Episode 1: Introduction to RSS</title>
      <link>https://example.com/podcast/episode-1</link>
      <description>In this episode, we discuss RSS feeds.</description>
      <pubDate>Wed, 13 Feb 2026 10:00:00 +0000</pubDate>
      <guid isPermaLink="true">https://example.com/podcast/episode-1</guid>

      <!-- Audio file -->
      <enclosure url="https://example.com/audio/episode-1.mp3"
                 length="24986239"
                 type="audio/mpeg"/>

      <itunes:duration>41:23</itunes:duration>
      <itunes:explicit>no</itunes:explicit>
      <itunes:subtitle>Introduction to RSS feeds</itunes:subtitle>
      <itunes:summary>Detailed episode description here.</itunes:summary>
    </item>
  </channel>
</rss>
```

**Key podcast elements:**
- `<enclosure>`: Audio/video file URL, size, and type
- `<itunes:*>`: iTunes-specific metadata
- `<itunes:image>`: Podcast artwork (1400×1400 minimum)
- `<itunes:duration>`: Episode length (HH:MM:SS)

## Feed Limits

**Number of items:** 10-20 recent items (standard practice)

**Full content:** Consider bandwidth and loading time

```go-html-template
{{/* Limit to 20 most recent items */}}
{{ range first 20 .Pages }}
<item>
  <!-- ... -->
</item>
{{ end }}
```

## HTML Link in RSS Feed

```html
<!-- Include clickable links in RSS description -->
<description>
  <![CDATA[
    <p>Article summary with <a href="https://example.com/link">clickable links</a>.</p>
    <p>Read more at <a href="https://example.com/blog/article">https://example.com/blog/article</a></p>
  ]]>
</description>
```

## Testing RSS Feeds

### Validators

**W3C Feed Validation Service:** https://validator.w3.org/feed/

**How to use:**
1. Enter feed URL
2. Click "Check"
3. Review errors and warnings

### Feed Readers

**Test in actual feed readers:**
- Feedly: https://feedly.com
- The Old Reader: https://theoldreader.com
- NewsBlur: https://newsblur.com

### Command Line

```bash
# Download feed
curl https://example.com/index.xml

# Validate XML syntax
curl https://example.com/index.xml | xmllint --format -

# Count items
curl https://example.com/index.xml | grep -c "<item>"

# Check feed URL is accessible
curl -I https://example.com/index.xml
# Should return: Content-Type: application/rss+xml
```

## S3 / CloudFront Deployment

```bash
# Upload RSS feed with correct MIME type
aws s3 cp public/index.xml s3://your-bucket/index.xml \
  --content-type "application/rss+xml; charset=utf-8" \
  --cache-control "public, max-age=3600"
```

**Terraform:**

```hcl
resource "aws_s3_bucket_object" "rss" {
  bucket       = aws_s3_bucket.website.id
  key          = "index.xml"
  source       = "public/index.xml"
  content_type = "application/rss+xml; charset=utf-8"
  cache_control = "public, max-age=3600"  # 1 hour
  etag         = filemd5("public/index.xml")
}
```

## NGINX Configuration

```nginx
# Serve RSS/Atom feeds with correct MIME type
location ~ \.(xml|rss|atom)$ {
    root /var/www/html;

    # Set MIME type
    types {
        application/rss+xml  rss xml;
        application/atom+xml atom;
    }

    # Cache for 1 hour
    add_header Cache-Control "public, max-age=3600";

    # CORS
    add_header Access-Control-Allow-Origin "*";

    charset utf-8;
}
```

## Common Mistakes

❌ **Invalid XML:**
```xml
<!-- Wrong - unescaped ampersand -->
<title>Tips & Tricks</title>

<!-- Correct -->
<title>Tips &amp; Tricks</title>
```

❌ **Wrong date format:**
```xml
<!-- Wrong (not RFC 822) -->
<pubDate>2026-02-13</pubDate>

<!-- Correct (RFC 822) -->
<pubDate>Wed, 13 Feb 2026 10:00:00 +0000</pubDate>
```

❌ **Missing required elements:**
```xml
<!-- Wrong - missing <link> and <description> -->
<channel>
  <title>My Blog</title>
</channel>

<!-- Correct -->
<channel>
  <title>My Blog</title>
  <link>https://example.com/</link>
  <description>Blog description</description>
</channel>
```

❌ **Relative URLs:**
```xml
<!-- Wrong -->
<link>/blog/post</link>

<!-- Correct -->
<link>https://example.com/blog/post</link>
```

❌ **No feed discovery link:**
```html
<!-- Wrong - no RSS link in HTML -->
<head>
  <title>Page</title>
</head>

<!-- Correct -->
<head>
  <title>Page</title>
  <link rel="alternate" type="application/rss+xml" href="/index.xml">
</head>
```

## Best Practices

**Required:**
- Valid XML syntax
- Absolute URLs
- RSS channel: title, link, description
- RSS item: title or description
- Correct MIME type (`application/rss+xml`)

**Recommended:**
- Feed discovery link in HTML
- Limit to 10-20 recent items
- Include `<pubDate>` for each item
- Include `<guid>` (permalink)
- Self-referencing `<atom:link rel="self">`

**Content Strategy:**
- Summary only (drives traffic to site)
- OR full content (convenient for readers)
- Include read-more link if using summary

**Update Frequency:**
- Update feed when new content published
- Cache feed (1-6 hours)
- Don't change items after publication

## Feed Icons

```html
<!-- RSS icon in header/footer -->
<a href="/index.xml" title="Subscribe to RSS">
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24">
    <path d="M6.18 15.64c-.83 0-1.5.67-1.5 1.5s.67 1.5 1.5 1.5 1.5-.67 1.5-1.5-.67-1.5-1.5-1.5zM4 4.44v2.83c7.03 0 12.73 5.7 12.73 12.73h2.83c0-8.59-6.97-15.56-15.56-15.56zm0 5.66v2.83c3.9 0 7.07 3.17 7.07 7.07h2.83c0-5.47-4.43-9.9-9.9-9.9z"/>
  </svg>
  RSS
</a>
```

## Hugo Complete RSS Example

```go-html-template
{{/* layouts/_default/rss.xml */}}
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>{{ if eq .Title .Site.Title }}{{ .Site.Title }}{{ else }}{{ .Title }} | {{ .Site.Title }}{{ end }}</title>
    <link>{{ .Permalink }}</link>
    <description>{{ with .Description }}{{ . }}{{ else }}{{ .Site.Params.description }}{{ end }}</description>
    <language>{{ .Site.LanguageCode | default "en-us" }}</language>
    {{ with .Site.Copyright }}<copyright>{{ . }}</copyright>{{ end }}
    <lastBuildDate>{{ now.Format "Mon, 02 Jan 2006 15:04:05 -0700" }}</lastBuildDate>
    <atom:link href="{{ .Permalink }}" rel="self" type="application/rss+xml" />

    {{ $pages := .Pages }}
    {{ if .IsHome }}
      {{ $pages = where site.RegularPages "Type" "in" site.Params.mainSections }}
    {{ end }}

    {{ range first 20 $pages }}
    <item>
      <title>{{ .Title }}</title>
      <link>{{ .Permalink }}</link>
      <pubDate>{{ .Date.Format "Mon, 02 Jan 2006 15:04:05 -0700" }}</pubDate>
      <guid isPermaLink="true">{{ .Permalink }}</guid>
      <description>
        <![CDATA[
          {{- with .Description }}{{ . }}{{ else }}{{ .Summary }}{{ end -}}
        ]]>
      </description>

      {{ with .Site.Author.email }}<author>{{ . }} ({{ $.Site.Author.name }})</author>{{ end }}

      {{ range (.GetTerms "categories") }}
      <category>{{ .LinkTitle }}</category>
      {{ end }}

      {{ range (.GetTerms "tags") }}
      <category>{{ .LinkTitle }}</category>
      {{ end }}
    </item>
    {{ end }}
  </channel>
</rss>
```

## Guidelines

**Essential:**
- Valid XML
- Absolute URLs
- Required channel/item elements
- Correct MIME type

**Recommended:**
- 10-20 recent items
- Feed discovery link in HTML
- Self-referencing atom:link
- Include pubDate and guid

**Best Practices:**
- Update when content changes
- Cache feed (1-6 hours)
- Test with validators and feed readers
- Decide: summary vs full content

## Benefits

Syndicated. Users subscribe to updates.

Automated. Feed readers fetch new content.

Portable. Standard format across platforms.

Timely. Readers notified of new posts.

## Related

- [sitemap-xml-advanced.md](./sitemap-xml-advanced.md) - XML sitemaps
- [meta-tags-seo.md](./meta-tags-seo.md) - Feed discovery link
- [open-graph-protocol.md](./open-graph-protocol.md) - Social metadata
