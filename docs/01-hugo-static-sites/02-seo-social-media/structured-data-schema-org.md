# Structured Data (Schema.org)

Schema.org structured data. JSON-LD. Rich snippets. Google Search enhancement. Semantic markup.

## Principle

Use Schema.org structured data to help search engines understand your content. Provide semantic markup in JSON-LD format. Enable rich snippets in search results. Improve SEO and click-through rates.

## What is Structured Data?

**Structured Data:** Machine-readable information about webpage content

**Purpose:** Help search engines understand context, relationships, and meaning

**Benefits:**
- Rich snippets in Google Search (star ratings, images, prices)
- Enhanced search appearance
- Better crawling and indexing
- Voice search optimization
- Knowledge Graph integration

**Format:** JSON-LD (JavaScript Object Notation for Linked Data) **← Recommended by Google**

**Specification:** https://schema.org/

## Why JSON-LD?

**Three markup formats available:**

1. **JSON-LD** (Recommended)
   - ✅ Separate from HTML
   - ✅ Easy to generate dynamically
   - ✅ No DOM manipulation
   - ✅ Easier to maintain
   - ✅ Google's preferred format

2. **Microdata** (Legacy)
   - ❌ Mixed with HTML
   - ❌ Harder to maintain
   - ❌ Harder to generate dynamically

3. **RDFa** (Legacy)
   - ❌ Mixed with HTML attributes
   - ❌ Complex syntax
   - ❌ Harder to debug

**Always use JSON-LD for new implementations.**

## Basic JSON-LD Structure

**JSON-LD block goes in `<head>` or `<body>` (before closing `</body>`):**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "10 Essential Hugo Performance Tips",
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  },
  "datePublished": "2026-02-13T10:00:00Z",
  "image": "https://example.com/images/article-image.jpg"
}
</script>
```

**Components:**

- `@context`: Always `"https://schema.org"`
- `@type`: Schema.org type (Article, Person, Organization, etc.)
- Properties: Type-specific fields

## Common Schema Types

### Article

**For blog posts, news articles, and written content.**

**Minimal Article:**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "10 Essential Hugo Performance Tips",
  "image": "https://example.com/images/hugo-performance.jpg",
  "datePublished": "2026-02-13T10:00:00Z",
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  }
}
</script>
```

**Complete Article:**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "10 Essential Hugo Performance Tips",
  "alternativeHeadline": "Make Your Hugo Site 3x Faster",
  "image": [
    "https://example.com/images/hugo-performance-1200x628.jpg",
    "https://example.com/images/hugo-performance-1200x1200.jpg",
    "https://example.com/images/hugo-performance-1200x900.jpg"
  ],
  "datePublished": "2026-02-13T10:00:00Z",
  "dateModified": "2026-02-14T09:15:00Z",
  "author": {
    "@type": "Person",
    "name": "Jane Doe",
    "url": "https://example.com/authors/jane-doe/"
  },
  "publisher": {
    "@type": "Organization",
    "name": "Hugo Best Practices",
    "logo": {
      "@type": "ImageObject",
      "url": "https://example.com/images/logo.png",
      "width": 600,
      "height": 60
    }
  },
  "description": "Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples.",
  "articleBody": "Full article text...",
  "wordCount": 2500,
  "keywords": ["Hugo", "Performance", "Static Sites", "Web Development"],
  "articleSection": "Web Development",
  "inLanguage": "en-US",
  "isAccessibleForFree": true,
  "mainEntityOfPage": {
    "@type": "WebPage",
    "@id": "https://example.com/blog/hugo-performance-tips/"
  }
}
</script>
```

**Required fields (Google Rich Results):**
- `headline` (max 110 characters)
- `image` (1200×630, 1200×1200, or 1200×900)
- `datePublished` (ISO 8601 format)
- `author` (Person or Organization)

**Recommended fields:**
- `dateModified`
- `publisher` (Organization with logo)
- `description`

### BlogPosting

**Subtype of Article specifically for blog posts.**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BlogPosting",
  "headline": "10 Essential Hugo Performance Tips",
  "image": "https://example.com/images/hugo-performance.jpg",
  "datePublished": "2026-02-13T10:00:00Z",
  "dateModified": "2026-02-14T09:15:00Z",
  "author": {
    "@type": "Person",
    "name": "Jane Doe",
    "url": "https://example.com/authors/jane-doe/"
  },
  "publisher": {
    "@type": "Organization",
    "name": "Hugo Best Practices",
    "logo": {
      "@type": "ImageObject",
      "url": "https://example.com/images/logo.png"
    }
  },
  "description": "Learn 10 proven techniques to make your Hugo site load 3x faster",
  "mainEntityOfPage": "https://example.com/blog/hugo-performance-tips/"
}
</script>
```

### Person

**For author pages and profiles.**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Person",
  "name": "Jane Doe",
  "url": "https://example.com/authors/jane-doe/",
  "image": "https://example.com/images/authors/jane-doe.jpg",
  "jobTitle": "Web Developer",
  "worksFor": {
    "@type": "Organization",
    "name": "Hugo Best Practices"
  },
  "sameAs": [
    "https://twitter.com/janedoe",
    "https://github.com/janedoe",
    "https://linkedin.com/in/janedoe"
  ],
  "description": "Web developer and technical writer specializing in Hugo and static sites"
}
</script>
```

### Organization

**For company/organization pages.**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "Hugo Best Practices",
  "url": "https://example.com",
  "logo": "https://example.com/images/logo.png",
  "description": "Resources and tutorials for building fast static sites with Hugo",
  "foundingDate": "2020-01-15",
  "founders": [
    {
      "@type": "Person",
      "name": "Jane Doe"
    }
  ],
  "contactPoint": {
    "@type": "ContactPoint",
    "contactType": "Customer Service",
    "email": "hello@example.com",
    "url": "https://example.com/contact/"
  },
  "sameAs": [
    "https://twitter.com/hugobest",
    "https://github.com/hugobest",
    "https://facebook.com/hugobest"
  ]
}
</script>
```

### WebSite

**For homepage and site-wide search.**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": "Hugo Best Practices",
  "url": "https://example.com",
  "description": "Learn to build fast static sites with Hugo",
  "publisher": {
    "@type": "Organization",
    "name": "Hugo Best Practices"
  },
  "potentialAction": {
    "@type": "SearchAction",
    "target": {
      "@type": "EntryPoint",
      "urlTemplate": "https://example.com/search?q={search_term_string}"
    },
    "query-input": "required name=search_term_string"
  }
}
</script>
```

**SearchAction enables Google Sitelinks Search Box** (search bar in Google results).

### BreadcrumbList

**For breadcrumb navigation (improves search results display).**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    {
      "@type": "ListItem",
      "position": 1,
      "name": "Home",
      "item": "https://example.com/"
    },
    {
      "@type": "ListItem",
      "position": 2,
      "name": "Blog",
      "item": "https://example.com/blog/"
    },
    {
      "@type": "ListItem",
      "position": 3,
      "name": "Hugo Performance Tips",
      "item": "https://example.com/blog/hugo-performance-tips/"
    }
  ]
}
</script>
```

**Google displays breadcrumbs in search results** when this is present.

### ItemList

**For blog post lists, product catalogs, etc.**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "ItemList",
  "itemListElement": [
    {
      "@type": "ListItem",
      "position": 1,
      "url": "https://example.com/blog/post1/"
    },
    {
      "@type": "ListItem",
      "position": 2,
      "url": "https://example.com/blog/post2/"
    },
    {
      "@type": "ListItem",
      "position": 3,
      "url": "https://example.com/blog/post3/"
    }
  ]
}
</script>
```

## Hugo Implementation

### Article Template

**layouts/partials/structured-data/article.html:**

```go-html-template
{{- $iso8601 := "2006-01-02T15:04:05Z07:00" -}}
{{- $images := slice -}}

{{- with .Params.images -}}
  {{- $images = . -}}
{{- else -}}
  {{- with $.Site.Params.og_image -}}
    {{- $images = slice . -}}
  {{- end -}}
{{- end -}}

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": {{ if eq .Section "blog" }}"BlogPosting"{{ else }}"Article"{{ end }},
  "headline": {{ .Title }},
  {{ with .Params.description }}
  "description": {{ . }},
  {{ end }}
  {{ with $images }}
  "image": {{ if gt (len .) 1 }}{{ . | jsonify }}{{ else }}{{ index . 0 }}{{ end }},
  {{ end }}
  "datePublished": {{ .Date.Format $iso8601 }},
  {{ if ne .Lastmod .Date }}
  "dateModified": {{ .Lastmod.Format $iso8601 }},
  {{ end }}
  "author": {
    "@type": "Person",
    "name": {{ if .Params.author }}{{ .Params.author }}{{ else }}{{ .Site.Params.author }}{{ end }},
    "url": "{{ .Site.BaseURL }}authors/{{ if .Params.author }}{{ .Params.author | urlize }}{{ else }}{{ .Site.Params.author | urlize }}{{ end }}/"
  },
  "publisher": {
    "@type": "Organization",
    "name": {{ .Site.Title }},
    "logo": {
      "@type": "ImageObject",
      "url": "{{ .Site.Params.logo | absURL }}"
    }
  },
  {{ with .Params.keywords }}
  "keywords": {{ . | jsonify }},
  {{ end }}
  {{ with .WordCount }}
  "wordCount": {{ . }},
  {{ end }}
  "mainEntityOfPage": {
    "@type": "WebPage",
    "@id": {{ .Permalink }}
  }
}
</script>
```

### Person Template

**layouts/partials/structured-data/person.html:**

```go-html-template
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Person",
  "name": {{ .Title }},
  "url": {{ .Permalink }},
  {{ with .Params.image }}
  "image": {{ . | absURL }},
  {{ end }}
  {{ with .Params.job_title }}
  "jobTitle": {{ . }},
  {{ end }}
  {{ with .Params.works_for }}
  "worksFor": {
    "@type": "Organization",
    "name": {{ . }}
  },
  {{ end }}
  {{ with .Params.social_profiles }}
  "sameAs": {{ . | jsonify }},
  {{ end }}
  "description": {{ .Description }}
}
</script>
```

### Organization Template

**layouts/partials/structured-data/organization.html:**

```go-html-template
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": {{ .Site.Title }},
  "url": {{ .Site.BaseURL }},
  "logo": {{ .Site.Params.logo | absURL }},
  "description": {{ .Site.Params.description }},
  {{ with .Site.Params.founding_date }}
  "foundingDate": {{ . }},
  {{ end }}
  {{ with .Site.Params.contact_email }}
  "contactPoint": {
    "@type": "ContactPoint",
    "contactType": "Customer Service",
    "email": {{ . }},
    "url": "{{ $.Site.BaseURL }}contact/"
  },
  {{ end }}
  {{ with .Site.Params.social_profiles }}
  "sameAs": {{ . | jsonify }}
  {{ end }}
}
</script>
```

### WebSite Template

**layouts/partials/structured-data/website.html:**

```go-html-template
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": {{ .Site.Title }},
  "url": {{ .Site.BaseURL }},
  "description": {{ .Site.Params.description }},
  "publisher": {
    "@type": "Organization",
    "name": {{ .Site.Title }}
  }{{ if .Site.Params.search_enabled }},
  "potentialAction": {
    "@type": "SearchAction",
    "target": {
      "@type": "EntryPoint",
      "urlTemplate": "{{ .Site.BaseURL }}search?q={search_term_string}"
    },
    "query-input": "required name=search_term_string"
  }
  {{ end }}
}
</script>
```

### BreadcrumbList Template

**layouts/partials/structured-data/breadcrumb.html:**

```go-html-template
{{- if gt (len .Ancestors) 0 -}}
{{- $ancestors := .Ancestors.Reverse -}}
{{- $position := 1 -}}
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    {{- range $index, $ancestor := $ancestors -}}
    {{- if $index }},{{ end -}}
    {
      "@type": "ListItem",
      "position": {{ add $index 1 }},
      "name": {{ $ancestor.Title }},
      "item": {{ $ancestor.Permalink }}
    }
    {{- end -}}
    ,{
      "@type": "ListItem",
      "position": {{ add (len $ancestors) 1 }},
      "name": {{ .Title }},
      "item": {{ .Permalink }}
    }
  ]
}
</script>
{{- end -}}
```

### Include in Layout

**layouts/_default/baseof.html:**

```go-html-template
<!DOCTYPE html>
<html lang="{{ .Site.Language.Lang }}">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{{ .Title }}</title>

  {{/* Open Graph and Twitter Cards */}}
  {{ partial "head/opengraph.html" . }}
  {{ partial "head/twitter-card.html" . }}

  {{/* Structured Data */}}
  {{ if .IsHome }}
    {{ partial "structured-data/website.html" . }}
    {{ partial "structured-data/organization.html" . }}
  {{ else if eq .Type "author" }}
    {{ partial "structured-data/person.html" . }}
  {{ else if or (eq .Type "blog") (eq .Type "post") }}
    {{ partial "structured-data/article.html" . }}
    {{ partial "structured-data/breadcrumb.html" . }}
  {{ else }}
    {{ partial "structured-data/breadcrumb.html" . }}
  {{ end }}
</head>
<body>
  {{ block "main" . }}{{ end }}
</body>
</html>
```

### Conditional Schema Types

**Dynamic type selection:**

```go-html-template
{{ $schemaType := "Article" }}

{{ if eq .Section "blog" }}
  {{ $schemaType = "BlogPosting" }}
{{ else if eq .Section "news" }}
  {{ $schemaType = "NewsArticle" }}
{{ else if eq .Section "tutorial" }}
  {{ $schemaType = "TechArticle" }}
{{ end }}

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": {{ $schemaType }},
  ...
}
</script>
```

## Content Front Matter

**Example front matter for blog post:**

```yaml
---
title: "10 Essential Hugo Performance Tips"
description: "Learn 10 proven techniques to make your Hugo site load 3x faster"
date: 2026-02-13T10:00:00Z
lastmod: 2026-02-14T09:15:00Z
author: Jane Doe
images:
  - /images/blog/hugo-performance.jpg
keywords:
  - Hugo
  - Performance
  - Static Sites
  - Web Development
---
```

**Example front matter for author page:**

```yaml
---
title: "Jane Doe"
description: "Web developer and technical writer"
image: /images/authors/jane-doe.jpg
job_title: "Web Developer"
works_for: "Hugo Best Practices"
social_profiles:
  - https://twitter.com/janedoe
  - https://github.com/janedoe
  - https://linkedin.com/in/janedoe
---
```

## Site Configuration

**config.toml:**

```toml
[params]
  author = "Hugo Best Practices"
  description = "Learn to build fast static sites with Hugo"
  logo = "/images/logo.png"
  og_image = "/images/og-default.jpg"

  # Organization
  founding_date = "2020-01-15"
  contact_email = "hello@example.com"

  # Social profiles
  social_profiles = [
    "https://twitter.com/hugobest",
    "https://github.com/hugobest",
    "https://facebook.com/hugobest"
  ]

  # Search
  search_enabled = true
```

## Testing Structured Data

### Google Rich Results Test

**URL:** https://search.google.com/test/rich-results

**How to use:**
1. Enter page URL or paste HTML
2. Click "Test URL"
3. View detected structured data
4. Check for errors and warnings
5. Preview rich results

**What to check:**
- ✅ All required fields present
- ✅ No errors (red)
- ✅ Resolve warnings (yellow)
- ✅ Valid image URLs
- ✅ Correct dates (ISO 8601)

### Schema Markup Validator

**URL:** https://validator.schema.org/

**How to use:**
1. Paste JSON-LD code
2. Click "Validate"
3. Check for syntax errors
4. Verify structure

### Google Search Console

**URL:** https://search.google.com/search-console

**Check coverage:**
1. Go to "Enhancements"
2. View "Articles" or other rich result types
3. Check valid pages
4. Fix errors and warnings

### Manual Validation

```bash
# Extract JSON-LD from page
curl -s https://example.com/blog/post/ | grep -o '<script type="application/ld+json">.*</script>'

# Validate JSON syntax
curl -s https://example.com/blog/post/ | grep -o '<script type="application/ld+json">.*</script>' | sed 's/<[^>]*>//g' | jq .
```

**Check for:**
- ✅ Valid JSON syntax
- ✅ All required fields
- ✅ Correct @context (https://schema.org)
- ✅ Correct @type
- ✅ Valid URLs (absolute, HTTPS)
- ✅ ISO 8601 dates

## Common Issues

### Invalid JSON

**Problem:** Syntax errors in JSON-LD

**Causes:**
- Missing commas
- Trailing commas
- Unescaped quotes
- Invalid characters

**Solution:**

```html
<!-- ❌ BAD: Trailing comma -->
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Title",
}

<!-- ✅ GOOD: No trailing comma -->
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Title"
}
```

**Test JSON syntax:**
```bash
cat structured-data.json | jq .
```

### Missing Required Fields

**Problem:** Google Rich Results Test shows errors

**Solution:**

```html
<!-- Article MUST have: -->
- headline
- image (1200×630, 1200×1200, or 1200×900)
- datePublished (ISO 8601)
- author (Person or Organization)

<!-- Example: -->
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Article Title",
  "image": "https://example.com/image-1200x630.jpg",
  "datePublished": "2026-02-13T10:00:00Z",
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  }
}
```

### Invalid Date Format

**Problem:** Dates not recognized

**Solution:**

```html
<!-- ❌ BAD -->
"datePublished": "2026-02-13"
"datePublished": "Feb 13, 2026"
"datePublished": "2026/02/13"

<!-- ✅ GOOD (ISO 8601) -->
"datePublished": "2026-02-13T10:00:00Z"
"datePublished": "2026-02-13T14:30:00-05:00"
```

### Image URL Issues

**Problem:** Image not displayed in rich results

**Solution:**

```html
<!-- ❌ BAD: Relative URL -->
"image": "/images/article.jpg"

<!-- ✅ GOOD: Absolute HTTPS URL -->
"image": "https://example.com/images/article.jpg"

<!-- ✅ GOOD: Multiple sizes (recommended) -->
"image": [
  "https://example.com/images/article-1200x628.jpg",
  "https://example.com/images/article-1200x1200.jpg",
  "https://example.com/images/article-1200x900.jpg"
]
```

### Logo Missing Width/Height

**Problem:** Publisher logo warning

**Solution:**

```html
<!-- ❌ BAD: No dimensions -->
"publisher": {
  "@type": "Organization",
  "name": "Publisher Name",
  "logo": "https://example.com/logo.png"
}

<!-- ✅ GOOD: With dimensions -->
"publisher": {
  "@type": "Organization",
  "name": "Publisher Name",
  "logo": {
    "@type": "ImageObject",
    "url": "https://example.com/logo.png",
    "width": 600,
    "height": 60
  }
}
```

## Best Practices

### General

**✅ DO:**
- Use JSON-LD (not Microdata or RDFa)
- Include all required fields
- Use absolute HTTPS URLs
- Use ISO 8601 date format
- Validate with Google Rich Results Test
- Test on staging before production
- Include multiple image sizes
- Keep JSON-LD in `<head>` (or before `</body>`)
- Use correct Schema.org type for content

**❌ DON'T:**
- Mix multiple formats (JSON-LD + Microdata)
- Use relative URLs
- Omit required fields
- Include HTML in JSON strings
- Add invisible content (spam)
- Use incorrect @type
- Skip validation testing

### Article Schema

**✅ DO:**
- Include headline (max 110 characters)
- Provide multiple image sizes (1200×628, 1200×1200, 1200×900)
- Use BlogPosting for blog posts
- Include author information
- Add dateModified if content updated
- Include publisher with logo
- Add keywords for topics
- Specify inLanguage

**❌ DON'T:**
- Exceed headline character limit
- Use images smaller than 1200px wide
- Omit author
- Forget publisher logo dimensions
- Use non-ISO date formats

### Image Requirements

**For Article/BlogPosting:**
- **Aspect ratios:** 16×9, 4×3, 1×1
- **Recommended sizes:**
  - 1200×628 (16×9)
  - 1200×1200 (1×1)
  - 1200×900 (4×3)
- **Minimum:** 1200px wide
- **File format:** JPG, PNG, WebP
- **Absolute URL:** HTTPS required

**For Organization logo:**
- **Aspect ratio:** 1×1 (square) or wide rectangle
- **Minimum:** 112×112 pixels
- **Recommended:** 600×60 pixels
- **Specify width and height**

### Date Format

**Always use ISO 8601:**

```html
<!-- ✅ GOOD -->
"datePublished": "2026-02-13T10:00:00Z"
"datePublished": "2026-02-13T14:30:00-05:00"
"datePublished": "2026-02-13T14:30:00+00:00"

<!-- ❌ BAD -->
"datePublished": "2026-02-13"
"datePublished": "Feb 13, 2026"
"datePublished": "13/02/2026"
```

## Advanced Techniques

### Multiple Schemas on One Page

**Combine different schema types:**

```html
<!-- Article schema -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Article Title",
  ...
}
</script>

<!-- BreadcrumbList schema -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [...]
}
</script>

<!-- Organization schema -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "Company Name",
  ...
}
</script>
```

**Or combine in single script using @graph:**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "Article",
      "headline": "Article Title",
      ...
    },
    {
      "@type": "BreadcrumbList",
      "itemListElement": [...]
    },
    {
      "@type": "Organization",
      "name": "Company Name",
      ...
    }
  ]
}
</script>
```

### Nested Schemas

**Reference other entities within schema:**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BlogPosting",
  "headline": "Article Title",
  "author": {
    "@type": "Person",
    "name": "Jane Doe",
    "sameAs": [
      "https://twitter.com/janedoe",
      "https://github.com/janedoe"
    ]
  },
  "publisher": {
    "@type": "Organization",
    "name": "Hugo Best Practices",
    "logo": {
      "@type": "ImageObject",
      "url": "https://example.com/logo.png",
      "width": 600,
      "height": 60
    }
  },
  "isPartOf": {
    "@type": "Blog",
    "@id": "https://example.com/blog/"
  }
}
</script>
```

### FAQ Schema

**For FAQ pages (enables FAQ rich results):**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "What is Hugo?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Hugo is a fast static site generator written in Go."
      }
    },
    {
      "@type": "Question",
      "name": "How fast is Hugo?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Hugo can build most sites in milliseconds, with build times under 1 second for typical blogs."
      }
    }
  ]
}
</script>
```

### HowTo Schema

**For tutorials and how-to guides:**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "HowTo",
  "name": "How to Install Hugo",
  "description": "Step-by-step guide to installing Hugo on your system",
  "image": "https://example.com/images/hugo-install.jpg",
  "totalTime": "PT10M",
  "step": [
    {
      "@type": "HowToStep",
      "name": "Download Hugo",
      "text": "Visit the Hugo releases page and download the appropriate binary",
      "url": "https://example.com/tutorial#step1"
    },
    {
      "@type": "HowToStep",
      "name": "Install Hugo",
      "text": "Extract the binary and move it to your PATH",
      "url": "https://example.com/tutorial#step2"
    },
    {
      "@type": "HowToStep",
      "name": "Verify Installation",
      "text": "Run 'hugo version' to verify the installation",
      "url": "https://example.com/tutorial#step3"
    }
  ]
}
</script>
```

### Review/Rating Schema

**For product reviews (enables star ratings in search):**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Hugo Pro Theme",
  "image": "https://example.com/images/hugo-pro.jpg",
  "description": "Premium Hugo theme with advanced features",
  "brand": {
    "@type": "Organization",
    "name": "Hugo Themes"
  },
  "offers": {
    "@type": "Offer",
    "price": "49.00",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock"
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.8",
    "reviewCount": "127"
  }
}
</script>
```

## Complete Examples

### Blog Post

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>10 Essential Hugo Performance Tips</title>

  <!-- Structured Data: Article -->
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "BlogPosting",
    "headline": "10 Essential Hugo Performance Tips",
    "image": [
      "https://example.com/images/hugo-perf-1200x628.jpg",
      "https://example.com/images/hugo-perf-1200x1200.jpg",
      "https://example.com/images/hugo-perf-1200x900.jpg"
    ],
    "datePublished": "2026-02-13T10:00:00Z",
    "dateModified": "2026-02-14T09:15:00Z",
    "author": {
      "@type": "Person",
      "name": "Jane Doe",
      "url": "https://example.com/authors/jane-doe/"
    },
    "publisher": {
      "@type": "Organization",
      "name": "Hugo Best Practices",
      "logo": {
        "@type": "ImageObject",
        "url": "https://example.com/images/logo.png",
        "width": 600,
        "height": 60
      }
    },
    "description": "Learn 10 proven techniques to make your Hugo site load 3x faster",
    "mainEntityOfPage": "https://example.com/blog/hugo-performance-tips/"
  }
  </script>

  <!-- Structured Data: Breadcrumb -->
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    "itemListElement": [
      {
        "@type": "ListItem",
        "position": 1,
        "name": "Home",
        "item": "https://example.com/"
      },
      {
        "@type": "ListItem",
        "position": 2,
        "name": "Blog",
        "item": "https://example.com/blog/"
      },
      {
        "@type": "ListItem",
        "position": 3,
        "name": "Hugo Performance Tips",
        "item": "https://example.com/blog/hugo-performance-tips/"
      }
    ]
  }
  </script>
</head>
<body>
  <!-- Content -->
</body>
</html>
```

### Homepage

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Hugo Best Practices</title>

  <!-- Structured Data: WebSite with SearchAction -->
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "WebSite",
    "name": "Hugo Best Practices",
    "url": "https://example.com",
    "description": "Learn to build fast static sites with Hugo",
    "publisher": {
      "@type": "Organization",
      "name": "Hugo Best Practices"
    },
    "potentialAction": {
      "@type": "SearchAction",
      "target": {
        "@type": "EntryPoint",
        "urlTemplate": "https://example.com/search?q={search_term_string}"
      },
      "query-input": "required name=search_term_string"
    }
  }
  </script>

  <!-- Structured Data: Organization -->
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Organization",
    "name": "Hugo Best Practices",
    "url": "https://example.com",
    "logo": "https://example.com/images/logo.png",
    "description": "Resources and tutorials for building fast static sites",
    "sameAs": [
      "https://twitter.com/hugobest",
      "https://github.com/hugobest"
    ]
  }
  </script>
</head>
<body>
  <!-- Content -->
</body>
</html>
```

## Guidelines

### Essential

**Required for Article rich results:**
- @context (https://schema.org)
- @type (Article, BlogPosting, or NewsArticle)
- headline (max 110 characters)
- image (1200×628, 1200×1200, or 1200×900)
- datePublished (ISO 8601)
- author (Person or Organization)

**Minimum viable implementation:**
```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Article Title",
  "image": "https://example.com/image-1200x628.jpg",
  "datePublished": "2026-02-13T10:00:00Z",
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  }
}
</script>
```

### Recommended

**For better rich results:**
- dateModified (if content updated)
- publisher (Organization with logo)
- description
- mainEntityOfPage
- keywords
- BreadcrumbList (separate schema)
- Multiple image sizes

### Advanced

**For maximum SEO benefit:**
- FAQ schema for FAQ pages
- HowTo schema for tutorials
- Review/Rating schema for products
- ItemList for post listings
- Multiple schemas with @graph
- Nested entities
- WebSite with SearchAction (homepage)

## Performance Checklist

**Before deploying:**

- [ ] Valid JSON syntax (test with jq)
- [ ] All required fields present
- [ ] Headline under 110 characters
- [ ] Images are 1200px+ wide
- [ ] Multiple image sizes provided (1200×628, 1200×1200, 1200×900)
- [ ] Dates in ISO 8601 format
- [ ] All URLs are absolute HTTPS
- [ ] Publisher logo includes width/height
- [ ] Tested with Google Rich Results Test
- [ ] No errors in Google Search Console
- [ ] Validated with Schema.org validator
- [ ] BreadcrumbList included (if applicable)
- [ ] Correct @type for content

## Benefits

Rich Snippets. Enhanced search results with images, ratings, dates.

Better CTR. Rich results increase click-through rates by 20-30%.

Voice Search. Structured data improves voice search optimization.

Knowledge Graph. Helps Google understand your content and entities.

SEO Boost. Better crawling, indexing, and ranking signals.

## Related

- [schema-org-types.md](./schema-org-types.md) - All Schema.org types reference
- [open-graph-meta-tags.md](./open-graph-meta-tags.md) - Social media metadata
- [meta-descriptions-titles.md](./meta-descriptions-titles.md) - Title and description optimization
- [sitemaps-robots-txt.md](./sitemaps-robots-txt.md) - Sitemap generation for crawling
- [canonical-urls.md](./canonical-urls.md) - Canonical URL specification
