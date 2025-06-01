# Structured Data (JSON-LD)

Machine-readable content metadata. Rich snippets in search results. Schema.org vocabulary. Google Search enhancement.

## Principle

Help search engines understand your content structure. Enable rich search results (rich snippets, knowledge panels). Improve SEO and click-through rates.

## What is JSON-LD?

**JSON-LD:** JavaScript Object Notation for Linked Data

**Purpose:** Embed structured data in web pages using JSON format

**Location:** `<script type="application/ld+json">` in HTML `<head>` or `<body>`

**Vocabulary:** Schema.org (standardized types and properties)

**Used By:**
- Google Search (rich results)
- Bing
- Yandex
- DuckDuckGo
- Social platforms

**Specification:** https://json-ld.org/
**Schema.org:** https://schema.org/

## Basic JSON-LD Example

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Complete Guide to Structured Data",
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  },
  "datePublished": "2026-02-13",
  "image": "https://example.com/article-image.jpg"
}
</script>
```

**Minimal required structure:**
- `@context`: Always `"https://schema.org"`
- `@type`: Schema.org type (Article, Product, Organization, etc.)
- Type-specific properties

## Common Schema Types

### Article (Blog Post)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "How to Implement JSON-LD Structured Data",
  "image": "https://example.com/images/json-ld-guide.jpg",
  "author": {
    "@type": "Person",
    "name": "Jane Doe",
    "url": "https://example.com/authors/jane-doe"
  },
  "publisher": {
    "@type": "Organization",
    "name": "Example Blog",
    "logo": {
      "@type": "ImageObject",
      "url": "https://example.com/logo.png"
    }
  },
  "datePublished": "2026-02-13T10:00:00Z",
  "dateModified": "2026-02-13T15:30:00Z",
  "description": "Complete guide to implementing JSON-LD structured data for better SEO."
}
</script>
```

### Organization

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "Example Company",
  "url": "https://example.com",
  "logo": "https://example.com/logo.png",
  "description": "Leading provider of web development tools",
  "contactPoint": {
    "@type": "ContactPoint",
    "telephone": "+1-555-123-4567",
    "contactType": "Customer Service",
    "areaServed": "US",
    "availableLanguage": "English"
  },
  "sameAs": [
    "https://twitter.com/example",
    "https://facebook.com/example",
    "https://linkedin.com/company/example"
  ]
}
</script>
```

### Person (Author Profile)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Person",
  "name": "Jane Doe",
  "url": "https://example.com/authors/jane-doe",
  "image": "https://example.com/authors/jane-doe.jpg",
  "jobTitle": "Senior Software Engineer",
  "worksFor": {
    "@type": "Organization",
    "name": "Example Company"
  },
  "sameAs": [
    "https://twitter.com/janedoe",
    "https://github.com/janedoe",
    "https://linkedin.com/in/janedoe"
  ]
}
</script>
```

### Website (Homepage)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": "Example Website",
  "url": "https://example.com",
  "description": "Your one-stop shop for web development resources",
  "publisher": {
    "@type": "Organization",
    "name": "Example Company"
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

### BreadcrumbList

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
      "item": "https://example.com"
    },
    {
      "@type": "ListItem",
      "position": 2,
      "name": "Blog",
      "item": "https://example.com/blog"
    },
    {
      "@type": "ListItem",
      "position": 3,
      "name": "Article Title",
      "item": "https://example.com/blog/article-slug"
    }
  ]
}
</script>
```

### Product (E-commerce)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Wireless Headphones XYZ",
  "image": "https://example.com/products/headphones.jpg",
  "description": "High-quality wireless headphones with noise cancellation",
  "sku": "WH-XYZ-2026",
  "brand": {
    "@type": "Brand",
    "name": "AudioTech"
  },
  "offers": {
    "@type": "Offer",
    "url": "https://example.com/products/headphones",
    "priceCurrency": "USD",
    "price": "199.99",
    "priceValidUntil": "2026-12-31",
    "availability": "https://schema.org/InStock",
    "seller": {
      "@type": "Organization",
      "name": "Example Store"
    }
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.5",
    "reviewCount": "127"
  }
}
</script>
```

### Event

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Event",
  "name": "Web Development Conference 2026",
  "startDate": "2026-06-15T09:00:00-07:00",
  "endDate": "2026-06-17T18:00:00-07:00",
  "eventStatus": "https://schema.org/EventScheduled",
  "eventAttendanceMode": "https://schema.org/OfflineEventAttendanceMode",
  "location": {
    "@type": "Place",
    "name": "Convention Center",
    "address": {
      "@type": "PostalAddress",
      "streetAddress": "123 Main Street",
      "addressLocality": "San Francisco",
      "addressRegion": "CA",
      "postalCode": "94105",
      "addressCountry": "US"
    }
  },
  "image": "https://example.com/events/webdev-conf.jpg",
  "description": "Annual web development conference featuring industry leaders",
  "offers": {
    "@type": "Offer",
    "url": "https://example.com/events/webdev-conf/register",
    "price": "299",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock",
    "validFrom": "2026-01-01T00:00:00Z"
  },
  "performer": {
    "@type": "Person",
    "name": "Jane Doe"
  },
  "organizer": {
    "@type": "Organization",
    "name": "WebDev Events",
    "url": "https://example.com"
  }
}
</script>
```

## Hugo Implementation

### Hugo Template (Article)

```go-html-template
{{/* layouts/partials/structured-data.html */}}

{{- if .IsPage }}
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": {{ .Title | jsonify }},
  "image": {{ (.Params.image | default "/images/default.jpg" | absURL) | jsonify }},
  "author": {
    "@type": "Person",
    "name": {{ (.Params.author | default .Site.Params.author.name) | jsonify }}
    {{- with .Params.authorURL }},
    "url": {{ . | absURL | jsonify }}
    {{- end }}
  },
  "publisher": {
    "@type": "Organization",
    "name": {{ .Site.Title | jsonify }},
    "logo": {
      "@type": "ImageObject",
      "url": {{ "/images/logo.png" | absURL | jsonify }}
    }
  },
  "datePublished": {{ .PublishDate.Format "2006-01-02T15:04:05Z07:00" | jsonify }},
  "dateModified": {{ .Lastmod.Format "2006-01-02T15:04:05Z07:00" | jsonify }},
  "description": {{ (.Description | default .Summary) | jsonify }},
  "mainEntityOfPage": {
    "@type": "WebPage",
    "@id": {{ .Permalink | jsonify }}
  }
}
</script>
{{- end }}
```

### Hugo Template (Organization)

```go-html-template
{{/* layouts/partials/organization-schema.html */}}

{{- if .IsHome }}
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": {{ .Site.Title | jsonify }},
  "url": {{ .Site.BaseURL | jsonify }},
  "logo": {{ "/images/logo.png" | absURL | jsonify }},
  "description": {{ .Site.Params.description | jsonify }}
  {{- with .Site.Params.social }},
  "sameAs": [
    {{- range $i, $social := . }}
    {{- if $i }},{{ end }}
    {{ $social | jsonify }}
    {{- end }}
  ]
  {{- end }}
}
</script>
{{- end }}
```

### Configuration (config.toml)

```toml
[params]
[params.author]
name = "Jane Doe"
url = "/authors/jane-doe"

[params.social]
twitter = "https://twitter.com/example"
github = "https://github.com/example"
linkedin = "https://linkedin.com/company/example"
```

### Content Front Matter

```yaml
---
title: "Complete Guide to JSON-LD"
date: 2026-02-13
description: "Learn how to implement structured data using JSON-LD."
author: "Jane Doe"
authorURL: "/authors/jane-doe"
image: "/images/json-ld-guide.jpg"
---
```

## Hugo Breadcrumb Schema

```go-html-template
{{/* layouts/partials/breadcrumb-schema.html */}}

{{- if not .IsHome }}
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    {
      "@type": "ListItem",
      "position": 1,
      "name": "Home",
      "item": {{ .Site.BaseURL | jsonify }}
    }
    {{- $sections := split .RelPermalink "/" }}
    {{- $path := "" }}
    {{- range $index, $section := $sections }}
      {{- if $section }}
        {{- $path = printf "%s/%s" $path $section }}
        {{- with $.Site.GetPage $path }},
    {
      "@type": "ListItem",
      "position": {{ add $index 2 }},
      "name": {{ .Title | jsonify }},
      "item": {{ .Permalink | jsonify }}
    }
        {{- end }}
      {{- end }}
    {{- end }}
  ]
}
</script>
{{- end }}
```

## Multiple Types on Same Page

```html
<!-- Article + Breadcrumb + Organization -->
<script type="application/ld+json">
[
  {
    "@context": "https://schema.org",
    "@type": "Article",
    "headline": "Article Title",
    "author": {
      "@type": "Person",
      "name": "Jane Doe"
    }
  },
  {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    "itemListElement": [...]
  },
  {
    "@context": "https://schema.org",
    "@type": "Organization",
    "name": "Example Company",
    "url": "https://example.com"
  }
]
</script>
```

**Or separate script tags:**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  ...
}
</script>

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  ...
}
</script>
```

## Testing Structured Data

### Google Rich Results Test

**URL:** https://search.google.com/test/rich-results

**How to use:**
1. Enter your URL or paste code
2. Click "Test URL"
3. Review results
4. Check for errors/warnings

### Schema.org Validator

**URL:** https://validator.schema.org/

**How to use:**
1. Paste your JSON-LD or URL
2. Click "Run Test"
3. Review validation results

### Google Search Console

**URL:** https://search.google.com/search-console

**Rich Results Report:**
- Shows indexed structured data
- Errors and warnings
- Performance of rich results

### Command Line Testing

```bash
# Extract JSON-LD from page
curl https://example.com | grep -A 50 'application/ld\+json'

# Validate JSON syntax
curl https://example.com | grep -A 50 'application/ld\+json' | jq '.'

# Test with Google's API
curl -X POST "https://searchconsole.googleapis.com/v1/urlTestingTools/mobileFriendlyTest:run" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com"}'
```

## Rich Results Types (Google)

**Common rich results enabled by JSON-LD:**

| Type | Description | Required Properties |
|------|-------------|---------------------|
| Article | News, blog posts | headline, image, datePublished, author |
| Breadcrumb | Navigation path | itemListElement |
| Product | E-commerce items | name, image, offers |
| Recipe | Cooking recipes | name, image, recipeIngredient |
| Event | Upcoming events | name, startDate, location |
| FAQ | Frequently asked questions | mainEntity (Question/Answer) |
| How-to | Step-by-step guides | step, tool, supply |
| Video | Video content | name, description, thumbnailUrl, uploadDate |
| Review | Product/service reviews | itemReviewed, reviewRating, author |

## FAQ Schema

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "What is JSON-LD?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "JSON-LD is a method of encoding linked data using JSON. It's used to add structured data to web pages."
      }
    },
    {
      "@type": "Question",
      "name": "Why use structured data?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Structured data helps search engines understand your content and can enable rich results in search."
      }
    }
  ]
}
</script>
```

## How-To Schema

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "HowTo",
  "name": "How to Implement JSON-LD",
  "description": "Step-by-step guide to adding structured data",
  "image": "https://example.com/how-to-image.jpg",
  "totalTime": "PT30M",
  "step": [
    {
      "@type": "HowToStep",
      "name": "Create JSON-LD script",
      "text": "Add a script tag with type application/ld+json",
      "url": "https://example.com/guide#step1"
    },
    {
      "@type": "HowToStep",
      "name": "Add schema markup",
      "text": "Include the appropriate Schema.org type and properties",
      "url": "https://example.com/guide#step2"
    },
    {
      "@type": "HowToStep",
      "name": "Test and validate",
      "text": "Use Google's Rich Results Test to validate your markup",
      "url": "https://example.com/guide#step3"
    }
  ]
}
</script>
```

## Video Schema

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "VideoObject",
  "name": "Introduction to JSON-LD",
  "description": "Learn the basics of JSON-LD structured data",
  "thumbnailUrl": "https://example.com/video-thumb.jpg",
  "uploadDate": "2026-02-13T10:00:00Z",
  "duration": "PT10M30S",
  "contentUrl": "https://example.com/videos/json-ld-intro.mp4",
  "embedUrl": "https://example.com/embed/video-123",
  "interactionStatistic": {
    "@type": "InteractionCounter",
    "interactionType": "https://schema.org/WatchAction",
    "userInteractionCount": 5647
  }
}
</script>
```

## Common Mistakes

❌ **Invalid JSON syntax:**
```html
<!-- Wrong (trailing comma) -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Title",
}
</script>

<!-- Correct -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Title"
}
</script>
```

❌ **Missing required properties:**
```html
<!-- Wrong (Article missing image, datePublished, author) -->
{
  "@type": "Article",
  "headline": "Title"
}

<!-- Correct -->
{
  "@type": "Article",
  "headline": "Title",
  "image": "https://example.com/image.jpg",
  "datePublished": "2026-02-13",
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  }
}
```

❌ **Relative URLs:**
```html
<!-- Wrong -->
"image": "/images/photo.jpg"

<!-- Correct -->
"image": "https://example.com/images/photo.jpg"
```

❌ **Wrong date format:**
```html
<!-- Wrong -->
"datePublished": "02/13/2026"

<!-- Correct (ISO 8601) -->
"datePublished": "2026-02-13T10:00:00Z"
```

❌ **Mismatched content:**
```html
<!-- Wrong - structured data doesn't match visible page content -->
<h1>Article About Dogs</h1>
<script type="application/ld+json">
{
  "headline": "Article About Cats"  // ❌ Mismatch!
}
</script>

<!-- Correct - structured data matches page -->
<h1>Article About Dogs</h1>
<script type="application/ld+json">
{
  "headline": "Article About Dogs"  // ✅ Match
}
</script>
```

## Best Practices

**Required:**
- Valid JSON syntax
- `@context` and `@type` always present
- Required properties for chosen type
- Absolute URLs for images and links

**Recommended:**
- Use most specific type available
- Include as many relevant properties as possible
- Match structured data to visible content
- Test with Google Rich Results Test
- Monitor Google Search Console for errors

**Optional:**
- Multiple schema types on same page
- Nested objects for detailed information
- Additional properties beyond required

**Validation:**
- Use Schema.org Validator
- Test with Google Rich Results Test
- Check Google Search Console
- Verify JSON syntax with linter

## Hugo Automation

### Auto-generate from Front Matter

```go-html-template
{{/* layouts/partials/auto-schema.html */}}

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": {{ .Params.schemaType | default "Article" | jsonify }},
  {{- if eq (.Params.schemaType | default "Article") "Article" }}
  "headline": {{ .Title | jsonify }},
  "image": {{ (.Params.image | default "/images/default.jpg" | absURL) | jsonify }},
  "datePublished": {{ .PublishDate.Format "2006-01-02T15:04:05Z07:00" | jsonify }},
  "dateModified": {{ .Lastmod.Format "2006-01-02T15:04:05Z07:00" | jsonify }},
  "author": {
    "@type": "Person",
    "name": {{ (.Params.author | default .Site.Params.author.name) | jsonify }}
  },
  "publisher": {
    "@type": "Organization",
    "name": {{ .Site.Title | jsonify }},
    "logo": {
      "@type": "ImageObject",
      "url": {{ "/images/logo.png" | absURL | jsonify }}
    }
  },
  "description": {{ (.Description | default .Summary | plainify | truncate 200) | jsonify }}
  {{- end }}
}
</script>
```

## Guidelines

**Essential:**
- `@context`: Always `"https://schema.org"`
- `@type`: Choose most specific type
- Required properties for chosen type
- Valid JSON syntax

**Recommended:**
- Test with Google Rich Results Test
- Include all relevant properties
- Use absolute URLs
- Match visible content

**Schema.org Types:**
- Article, BlogPosting, NewsArticle
- Organization, Person
- Product, Offer
- Event, WebSite
- BreadcrumbList
- FAQPage, HowTo
- VideoObject

## Benefits

Discoverable. Search engines understand your content.

Enhanced. Rich results in search (images, ratings, etc.).

Structured. Machine-readable metadata.

SEO. Better search visibility and CTR.

## Related

- [schema-org-markup.md](./schema-org-markup.md) - Schema.org types
- [open-graph-protocol.md](./open-graph-protocol.md) - Open Graph tags
- [meta-tags-seo.md](./meta-tags-seo.md) - HTML meta tags
- [sitemap-xml-advanced.md](./sitemap-xml-advanced.md) - XML sitemaps
