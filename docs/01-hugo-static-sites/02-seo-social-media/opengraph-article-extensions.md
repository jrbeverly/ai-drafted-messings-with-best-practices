# Open Graph Article Extensions

Article-specific Open Graph tags. Blog post metadata. Author attribution. Publication dates. Article sections.

## Principle

Use Open Graph article extensions for blog posts and news articles. Provide structured metadata about authors, publication dates, sections, and tags. Enable rich article previews on social platforms.

## Article Type

**Use article type for blog posts:**

```html
<meta property="og:type" content="article">
```

**Triggers article-specific tags**

## Article Tags

### article:published_time

```html
<meta property="article:published_time" content="2026-02-13T10:00:00Z">
```

**Format:** ISO 8601 datetime (YYYY-MM-DDTHH:MM:SSZ)

**Example:**
```html
<meta property="article:published_time" content="2026-02-13T14:30:00Z">
```

### article:modified_time

```html
<meta property="article:modified_time" content="2026-02-14T09:15:00Z">
```

**Purpose:** Last update timestamp

**Example:**
```html
<meta property="article:modified_time" content="2026-02-14T09:15:00Z">
```

### article:expiration_time

```html
<meta property="article:expiration_time" content="2027-02-13T00:00:00Z">
```

**Use case:** Time-sensitive content (event announcements, sales)

### article:author

```html
<meta property="article:author" content="https://facebook.com/authorprofile">
```

**Multiple authors:**
```html
<meta property="article:author" content="https://facebook.com/author1">
<meta property="article:author" content="https://facebook.com/author2">
```

**Purpose:** Links to author's Facebook profile

**Alternative:** Profile URL on your site
```html
<meta property="article:author" content="https://example.com/authors/jane-doe">
```

### article:section

```html
<meta property="article:section" content="Technology">
```

**Purpose:** High-level category/section

**Examples:**
```html
<meta property="article:section" content="Web Development">
<meta property="article:section" content="Tutorials">
<meta property="article:section" content="News">
```

**Note:** Only one section per article

### article:tag

```html
<meta property="article:tag" content="Hugo">
<meta property="article:tag" content="Static Sites">
<meta property="article:tag" content="Performance">
```

**Purpose:** Specific keywords/tags

**Multiple tags supported:**
```html
<meta property="article:tag" content="Hugo">
<meta property="article:tag" content="Go">
<meta property="article:tag" content="SSG">
<meta property="article:tag" content="JAMstack">
<meta property="article:tag" content="Web Performance">
```

## Complete Article Example

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>10 Hugo Performance Tips | Blog</title>

  <!-- Basic Open Graph -->
  <meta property="og:type" content="article">
  <meta property="og:url" content="https://example.com/blog/hugo-performance-tips/">
  <meta property="og:title" content="10 Essential Hugo Performance Tips">
  <meta property="og:description" content="Learn proven techniques to make your Hugo site blazing fast.">
  <meta property="og:image" content="https://example.com/images/blog/hugo-perf-og.jpg">
  <meta property="og:site_name" content="Hugo Best Practices">

  <!-- Article-specific tags -->
  <meta property="article:published_time" content="2026-02-13T10:00:00Z">
  <meta property="article:modified_time" content="2026-02-14T09:15:00Z">
  <meta property="article:author" content="https://example.com/authors/jane-doe">
  <meta property="article:section" content="Web Development">
  <meta property="article:tag" content="Hugo">
  <meta property="article:tag" content="Performance">
  <meta property="article:tag" content="Static Sites">
  <meta property="article:tag" content="Web Performance">
</head>
<body>
  <!-- Article content -->
</body>
</html>
```

## Hugo Implementation

### Hugo Template

**layouts/partials/head/opengraph-article.html:**

```go-html-template
{{ if eq .Type "blog" }}
  {{/* Article-specific tags */}}
  <meta property="og:type" content="article">

  {{/* Published time */}}
  {{ with .Date }}
    <meta property="article:published_time" content="{{ .Format "2006-01-02T15:04:05Z07:00" }}">
  {{ end }}

  {{/* Modified time */}}
  {{ with .Lastmod }}
    {{ if ne .Unix $.Date.Unix }}
      <meta property="article:modified_time" content="{{ .Format "2006-01-02T15:04:05Z07:00" }}">
    {{ end }}
  {{ end }}

  {{/* Expiration time */}}
  {{ with .Params.expiry_date }}
    <meta property="article:expiration_time" content="{{ . }}">
  {{ end }}

  {{/* Author */}}
  {{ with .Params.author }}
    {{ if reflect.IsSlice . }}
      {{ range . }}
        <meta property="article:author" content="{{ $.Site.BaseURL }}authors/{{ . | urlize }}">
      {{ end }}
    {{ else }}
      <meta property="article:author" content="{{ $.Site.BaseURL }}authors/{{ . | urlize }}">
    {{ end }}
  {{ end }}

  {{/* Section */}}
  {{ with .Section }}
    <meta property="article:section" content="{{ . | title }}">
  {{ end }}

  {{/* Tags */}}
  {{ with .Params.tags }}
    {{ range . }}
      <meta property="article:tag" content="{{ . }}">
    {{ end }}
  {{ end }}
{{ end }}
```

### Content Front Matter

```yaml
---
title: "10 Essential Hugo Performance Tips"
date: 2026-02-13T10:00:00Z
lastmod: 2026-02-14T09:15:00Z
author: Jane Doe
# or multiple authors:
# author:
#   - Jane Doe
#   - John Smith
section: Web Development
tags:
  - Hugo
  - Performance
  - Static Sites
  - Web Performance
expiry_date: 2027-02-13T00:00:00Z  # Optional
---
```

## Profile Tags (for Author Pages)

**Use profile type for author pages:**

```html
<meta property="og:type" content="profile">
<meta property="profile:first_name" content="Jane">
<meta property="profile:last_name" content="Doe">
<meta property="profile:username" content="janedoe">
<meta property="profile:gender" content="female">
```

**Complete author page:**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta property="og:type" content="profile">
  <meta property="og:url" content="https://example.com/authors/jane-doe/">
  <meta property="og:title" content="Jane Doe - Author">
  <meta property="og:image" content="https://example.com/images/authors/jane-doe.jpg">

  <meta property="profile:first_name" content="Jane">
  <meta property="profile:last_name" content="Doe">
  <meta property="profile:username" content="janedoe">
</head>
</html>
```

## Best Practices

### Dates

**✅ DO:**
- Use ISO 8601 format
- Include timezone (Z for UTC)
- Update `modified_time` on edits
- Use consistent timezone

**❌ DON'T:**
- Use relative dates ("yesterday")
- Omit timezone
- Use non-standard formats

**Examples:**
```html
<!-- ✅ GOOD -->
<meta property="article:published_time" content="2026-02-13T10:00:00Z">
<meta property="article:published_time" content="2026-02-13T14:30:00-05:00">

<!-- ❌ BAD -->
<meta property="article:published_time" content="2026-02-13">
<meta property="article:published_time" content="Feb 13, 2026">
```

### Authors

**✅ DO:**
- Link to author profile page
- Use consistent author URLs
- Support multiple authors
- Include all co-authors

**❌ DON'T:**
- Use author name as value (use URL)
- Link to external profiles only
- Omit author attribution

### Tags

**✅ DO:**
- Use 3-8 tags per article
- Use specific, relevant keywords
- Consistent capitalization (Title Case or lowercase)
- Include primary topic first

**❌ DON'T:**
- Overload with 20+ tags
- Use overly broad tags
- Duplicate information from title
- Include irrelevant keywords

### Sections

**✅ DO:**
- Use high-level category
- One section per article
- Match site navigation structure
- Use title case

**❌ DON'T:**
- Use multiple sections
- Use very specific categories (use tags instead)
- Change section structure frequently

## Hugo Automation

### Automatic Section from Directory

```go-html-template
{{ $section := .Section | title }}
{{ if eq .Section "blog" }}
  {{ $section = .Params.category | default "General" }}
{{ end }}
<meta property="article:section" content="{{ $section }}">
```

### Auto-generate Tags from Taxonomy

```go-html-template
{{ with .Params.tags }}
  {{ range first 8 . }}
    <meta property="article:tag" content="{{ . }}">
  {{ end }}
{{ end }}
```

### Multiple Author Support

```go-html-template
{{ with .Params.authors }}
  {{ range . }}
    {{ $authorSlug := . | urlize }}
    {{ $authorPage := $.Site.GetPage (printf "/authors/%s" $authorSlug) }}
    {{ if $authorPage }}
      <meta property="article:author" content="{{ $authorPage.Permalink }}">
    {{ else }}
      <meta property="article:author" content="{{ $.Site.BaseURL }}authors/{{ $authorSlug }}">
    {{ end }}
  {{ end }}
{{ end }}
```

## Testing

### Facebook Sharing Debugger

**URL:** https://developers.facebook.com/tools/debug/

**Check:**
- Article tags appear
- Dates formatted correctly
- Author links work
- Tags display properly

### Validation

```bash
# Check for article tags in HTML
curl -s https://example.com/blog/post/ | grep "article:"

# Validate date format
curl -s https://example.com/blog/post/ | grep -o 'content="[0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\}T[0-9]\{2\}:[0-9]\{2\}:[0-9]\{2\}.*"'
```

## Common Patterns

### Blog Post (Single Author)

```html
<meta property="og:type" content="article">
<meta property="article:published_time" content="2026-02-13T10:00:00Z">
<meta property="article:author" content="https://example.com/authors/jane-doe">
<meta property="article:section" content="Tutorials">
<meta property="article:tag" content="Hugo">
<meta property="article:tag" content="Static Sites">
```

### News Article (Multiple Authors)

```html
<meta property="og:type" content="article">
<meta property="article:published_time" content="2026-02-13T08:30:00Z">
<meta property="article:modified_time" content="2026-02-13T14:15:00Z">
<meta property="article:author" content="https://example.com/authors/reporter1">
<meta property="article:author" content="https://example.com/authors/reporter2">
<meta property="article:section" content="Tech News">
<meta property="article:tag" content="Breaking News">
<meta property="article:tag" content="Technology">
```

### Tutorial (with Expiration)

```html
<meta property="og:type" content="article">
<meta property="article:published_time" content="2026-02-13T10:00:00Z">
<meta property="article:expiration_time" content="2027-02-13T00:00:00Z">
<meta property="article:author" content="https://example.com/authors/instructor">
<meta property="article:section" content="Tutorials">
<meta property="article:tag" content="Hugo">
<meta property="article:tag" content="Beginner">
```

## Guidelines

**Essential:**
- og:type = "article"
- article:published_time
- article:author
- article:section

**Recommended:**
- article:modified_time (if updated)
- article:tag (3-8 tags)
- ISO 8601 date format

**Optional:**
- article:expiration_time (time-sensitive content)
- Multiple authors (co-authored posts)

## Benefits

Rich Metadata. Structured article information.

Author Attribution. Proper credit and linking.

Better Organization. Sections and tags for discovery.

Time Context. Publication and update dates visible.

## Related

- [open-graph-meta-tags.md](./open-graph-meta-tags.md) - Basic OG tags
- [structured-data-schema-org.md](./structured-data-schema-org.md) - Schema.org Article type
- [meta-descriptions-titles.md](./meta-descriptions-titles.md) - Title and description optimization
- [../../01-hugo-basics/hugo-taxonomies-sections.md](../01-hugo-basics/hugo-taxonomies-sections.md) - Content taxonomies
