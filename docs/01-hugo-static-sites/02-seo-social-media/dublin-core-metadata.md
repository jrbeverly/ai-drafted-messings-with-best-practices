# Dublin Core Metadata

Dublin Core elements. DC meta tags. Resource description. Library metadata standards. Hugo implementation.

## Principle

Use Dublin Core metadata to provide standardized resource description for academic, library, and archival contexts. Implement DC elements for enhanced discoverability in scholarly and government repositories.

## What is Dublin Core?

**Dublin Core:** Standardized set of metadata elements for describing resources

**Origin:** Dublin, Ohio metadata workshop (1995)

**Standard:** ISO 15836, IETF RFC 5013

**Purpose:**
- Describe any resource (web pages, documents, images)
- Cross-domain interoperability
- Library and archive cataloging
- Academic paper discovery
- Government document management

**Website:** https://www.dublincore.org/

## Dublin Core Elements

### The 15 Core Elements

| Element | Description | Example |
|---------|-------------|---------|
| dc.title | Resource title | "10 Hugo Performance Tips" |
| dc.creator | Primary author/creator | "Jane Doe" |
| dc.subject | Topic/keywords | "Hugo; Static Sites; Performance" |
| dc.description | Resource description | "Learn optimization techniques..." |
| dc.publisher | Entity publishing resource | "Hugo Best Practices" |
| dc.contributor | Other contributors | "John Smith" |
| dc.date | Publication date | "2026-02-13" |
| dc.type | Nature of resource | "Text" |
| dc.format | File format | "text/html" |
| dc.identifier | Unique identifier | URL, DOI, ISBN |
| dc.source | Source of derived resource | Original URL |
| dc.language | Language of resource | "en" |
| dc.relation | Related resource | Link to related page |
| dc.coverage | Spatial/temporal coverage | "Global" |
| dc.rights | Copyright/license | "CC BY 4.0" |

### HTML Meta Tag Syntax

```html
<meta name="dc.title" content="10 Essential Hugo Performance Tips">
<meta name="dc.creator" content="Jane Doe">
<meta name="dc.subject" content="Hugo; Static Sites; Web Performance">
<meta name="dc.description" content="Learn 10 proven techniques to optimize Hugo sites">
<meta name="dc.publisher" content="Hugo Best Practices">
<meta name="dc.date" content="2026-02-13">
<meta name="dc.type" content="Text">
<meta name="dc.format" content="text/html">
<meta name="dc.identifier" content="https://example.com/blog/hugo-performance-tips/">
<meta name="dc.language" content="en">
<meta name="dc.rights" content="Copyright 2026 Hugo Best Practices. All rights reserved.">
```

### Dublin Core Qualified (Extended)

**Refinements of core elements:**

```html
<!-- Date refinements -->
<meta name="dcterms.created" content="2026-02-13">
<meta name="dcterms.modified" content="2026-02-14">
<meta name="dcterms.issued" content="2026-02-13">

<!-- Coverage refinements -->
<meta name="dcterms.spatial" content="Global">
<meta name="dcterms.temporal" content="2026">

<!-- Rights refinements -->
<meta name="dcterms.license" content="https://creativecommons.org/licenses/by/4.0/">
<meta name="dcterms.accessRights" content="public">

<!-- Format refinements -->
<meta name="dcterms.extent" content="2500 words">
<meta name="dcterms.medium" content="online">

<!-- Relation refinements -->
<meta name="dcterms.isPartOf" content="Hugo Tutorial Series">
<meta name="dcterms.references" content="https://gohugo.io/documentation/">
```

## Hugo Implementation

### Basic Template

**layouts/partials/head/dublin-core.html:**

```go-html-template
{{/* Dublin Core Metadata */}}
<meta name="dc.title" content="{{ .Title }}">

{{ with .Params.author }}
  <meta name="dc.creator" content="{{ . }}">
{{ else }}
  {{ with .Site.Params.author }}
    <meta name="dc.creator" content="{{ . }}">
  {{ end }}
{{ end }}

{{ with .Params.tags }}
  <meta name="dc.subject" content="{{ delimit . "; " }}">
{{ end }}

{{ with .Description }}
  <meta name="dc.description" content="{{ . }}">
{{ else }}
  {{ with .Summary }}
    <meta name="dc.description" content="{{ . | plainify | truncate 200 }}">
  {{ end }}
{{ end }}

<meta name="dc.publisher" content="{{ .Site.Title }}">

{{ with .Params.contributors }}
  {{ range . }}
    <meta name="dc.contributor" content="{{ . }}">
  {{ end }}
{{ end }}

<meta name="dc.date" content="{{ .Date.Format "2006-01-02" }}">
<meta name="dc.type" content="{{ if .IsPage }}Text{{ else }}Collection{{ end }}">
<meta name="dc.format" content="text/html">
<meta name="dc.identifier" content="{{ .Permalink }}">
<meta name="dc.language" content="{{ .Site.Language.Lang }}">

{{ with .Site.Params.copyright }}
  <meta name="dc.rights" content="{{ . }}">
{{ end }}
```

### Extended Template (Dublin Core Terms)

**layouts/partials/head/dublin-core-extended.html:**

```go-html-template
{{/* Dublin Core Basic */}}
<meta name="dc.title" content="{{ .Title }}">

{{ with .Params.author }}
  <meta name="dc.creator" content="{{ . }}">
{{ else }}
  {{ with .Site.Params.author }}
    <meta name="dc.creator" content="{{ . }}">
  {{ end }}
{{ end }}

{{ with .Params.tags }}
  <meta name="dc.subject" content="{{ delimit . "; " }}">
{{ end }}

{{ with .Description }}
  <meta name="dc.description" content="{{ . }}">
{{ end }}

<meta name="dc.publisher" content="{{ .Site.Title }}">
<meta name="dc.format" content="text/html">
<meta name="dc.identifier" content="{{ .Permalink }}">
<meta name="dc.language" content="{{ .Site.Language.Lang }}">

{{/* Dublin Core Terms (Qualified) */}}
<meta name="dcterms.created" content="{{ .Date.Format "2006-01-02" }}">
{{ if ne .Lastmod .Date }}
  <meta name="dcterms.modified" content="{{ .Lastmod.Format "2006-01-02" }}">
{{ end }}
<meta name="dcterms.issued" content="{{ .Date.Format "2006-01-02" }}">

{{ with .WordCount }}
  <meta name="dcterms.extent" content="{{ . }} words">
{{ end }}

{{ with .Site.Params.license_url }}
  <meta name="dcterms.license" content="{{ . }}">
{{ end }}

<meta name="dcterms.accessRights" content="public">

{{ if .IsTranslated }}
  {{ range .Translations }}
    <meta name="dcterms.hasVersion" content="{{ .Permalink }}">
  {{ end }}
{{ end }}

{{ with .Section }}
  <meta name="dcterms.isPartOf" content="{{ . }}">
{{ end }}
```

### Include in Layout

**layouts/_default/baseof.html:**

```go-html-template
<head>
  <meta charset="utf-8">
  <title>{{ .Title }}</title>

  {{/* Standard meta tags */}}
  {{ partial "head/meta-tags.html" . }}

  {{/* Dublin Core */}}
  {{ partial "head/dublin-core.html" . }}
</head>
```

### Content Front Matter

```yaml
---
title: "10 Essential Hugo Performance Tips"
description: "Learn 10 proven techniques to optimize Hugo site performance"
date: 2026-02-13T10:00:00Z
lastmod: 2026-02-14T09:15:00Z
author: Jane Doe
contributors:
  - John Smith
  - Bob Johnson
tags:
  - Hugo
  - Performance
  - Static Sites
---
```

## Dublin Core in RSS Feeds

### RSS with DC Namespace

```xml
<rss version="2.0"
  xmlns:dc="http://purl.org/dc/elements/1.1/"
  xmlns:dcterms="http://purl.org/dc/terms/">
  <channel>
    <title>Hugo Best Practices</title>
    <item>
      <title>10 Hugo Performance Tips</title>
      <dc:creator>Jane Doe</dc:creator>
      <dc:date>2026-02-13T10:00:00Z</dc:date>
      <dc:subject>Hugo; Performance</dc:subject>
      <dc:type>Text</dc:type>
      <dc:language>en</dc:language>
      <dcterms.modified>2026-02-14T09:15:00Z</dcterms.modified>
    </item>
  </channel>
</rss>
```

## Common Use Cases

### Blog Post

```html
<meta name="dc.title" content="10 Essential Hugo Performance Tips">
<meta name="dc.creator" content="Jane Doe">
<meta name="dc.subject" content="Hugo; Static Sites; Web Performance">
<meta name="dc.description" content="Learn 10 proven techniques to optimize Hugo sites">
<meta name="dc.publisher" content="Hugo Best Practices">
<meta name="dc.date" content="2026-02-13">
<meta name="dc.type" content="Text">
<meta name="dc.format" content="text/html">
<meta name="dc.identifier" content="https://example.com/blog/hugo-tips/">
<meta name="dc.language" content="en">
```

### Academic Paper

```html
<meta name="dc.title" content="Performance Analysis of Static Site Generators">
<meta name="dc.creator" content="Dr. Jane Doe">
<meta name="dc.creator" content="Dr. John Smith">
<meta name="dc.subject" content="Static Site Generators; Web Performance; Benchmarking">
<meta name="dc.description" content="Comparative performance analysis of Hugo, Jekyll, and Gatsby">
<meta name="dc.publisher" content="Web Performance Journal">
<meta name="dc.date" content="2026-02-13">
<meta name="dc.type" content="Text">
<meta name="dc.format" content="application/pdf">
<meta name="dc.identifier" content="doi:10.1234/example.2026.001">
<meta name="dc.language" content="en">
<meta name="dc.rights" content="CC BY 4.0">
<meta name="dcterms.bibliographicCitation" content="Doe, J. & Smith, J. (2026). Performance Analysis...">
```

### Government Document

```html
<meta name="dc.title" content="Digital Services Accessibility Report 2026">
<meta name="dc.creator" content="Department of Digital Services">
<meta name="dc.subject" content="Accessibility; WCAG; Government">
<meta name="dc.description" content="Annual report on digital accessibility compliance">
<meta name="dc.publisher" content="Government Publishing Office">
<meta name="dc.date" content="2026-02-13">
<meta name="dc.type" content="Text">
<meta name="dc.format" content="text/html">
<meta name="dc.identifier" content="GOV-2026-ACCESSIBILITY-001">
<meta name="dc.language" content="en">
<meta name="dc.rights" content="Public Domain">
<meta name="dcterms.spatial" content="United States">
<meta name="dcterms.temporal" content="2025-2026">
```

## Best Practices

**✅ DO:**
- Include at minimum: title, creator, date, identifier, language
- Use ISO 8601 dates (YYYY-MM-DD)
- Use ISO 639-1 language codes
- Use semicolons to separate multiple subjects
- Use absolute URLs for identifiers
- Include dc.rights for licensing
- Use dcterms for refined metadata

**❌ DON'T:**
- Include empty Dublin Core tags
- Use ambiguous dates
- Skip language specification
- Use non-standard element names
- Duplicate content from other meta tags excessively

## Guidelines

### Essential

**Minimum Dublin Core:**
- dc.title
- dc.creator
- dc.date
- dc.identifier
- dc.language

### Recommended

**For better discoverability:**
- dc.subject (keywords)
- dc.description
- dc.publisher
- dc.type
- dc.format
- dc.rights
- dcterms.created / dcterms.modified

### Advanced

**For scholarly/archival use:**
- dcterms.bibliographicCitation
- dcterms.license
- dcterms.accessRights
- dcterms.spatial / dcterms.temporal
- dcterms.isPartOf
- Multiple dc.creator entries
- Dublin Core in RSS feeds

## Benefits

Interoperability. Universal metadata standard across domains.

Discoverability. Enhanced search in academic/library databases.

Standardization. ISO 15836 international standard.

Simplicity. Only 15 core elements to implement.

Compatibility. Works alongside Open Graph, Schema.org, and other standards.

## Related

- [structured-data-schema-org.md](./structured-data-schema-org.md) - Schema.org structured data
- [meta-tags-comprehensive.md](./meta-tags-comprehensive.md) - All HTML meta tags
- [rss-feeds.md](./rss-feeds.md) - Dublin Core in RSS feeds
- [open-graph-meta-tags.md](./open-graph-meta-tags.md) - Open Graph Protocol
