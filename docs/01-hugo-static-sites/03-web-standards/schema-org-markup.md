# Schema.org Markup

Schema.org vocabulary. Semantic web. Structured data types. Rich snippets. Search engine enhancement.

## Principle

Use standardized vocabulary to describe content. Help machines understand web pages. Enable rich search results. Improve discoverability.

## What is Schema.org?

**Schema.org:** Collaborative vocabulary for structured data markup

**Created By:** Google, Microsoft, Yahoo, Yandex (2011)

**Purpose:** Standardized types and properties for describing web content

**Format:** Can be implemented via:
- JSON-LD (recommended) - see [structured-data-json-ld.md](./structured-data-json-ld.md)
- Microdata (HTML attributes)
- RDFa (HTML attributes)

**Specification:** https://schema.org/

**This document focuses on Schema.org types and their properties**, not implementation format.

## Core Concepts

### Types

**Type:** A class of things (Article, Person, Product, etc.)

**Hierarchy:** Types inherit from parent types

```
Thing (root)
  └─ CreativeWork
      ├─ Article
      │   ├─ NewsArticle
      │   ├─ BlogPosting
      │   └─ TechArticle
      ├─ Book
      ├─ Movie
      └─ VideoObject
  └─ Organization
  └─ Person
  └─ Place
  └─ Product
  └─ Event
```

### Properties

**Property:** Attributes of a type

**Example:**
- `Article` has properties: `headline`, `datePublished`, `author`
- `Person` has properties: `name`, `email`, `jobTitle`

**Expected Types:** Properties expect specific types

```
Article
  author: Person or Organization
  datePublished: Date
  headline: Text
  image: ImageObject or URL
```

## Common Schema Types

### Article

**Description:** News articles, blog posts, scholarly papers

**Required Properties:**
- `headline`: Title of article
- `image`: Featured image
- `datePublished`: Publication date
- `author`: Author (Person or Organization)

**Recommended Properties:**
- `dateModified`: Last modified date
- `description`: Article summary
- `publisher`: Publishing organization

**Subtypes:**
- `NewsArticle`: News articles
- `BlogPosting`: Blog posts
- `TechArticle`: Technical articles
- `ScholarlyArticle`: Academic papers

**Example (JSON-LD):**

```json
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Complete Guide to Schema.org",
  "image": "https://example.com/article-image.jpg",
  "datePublished": "2026-02-13T10:00:00Z",
  "dateModified": "2026-02-13T15:30:00Z",
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
  "description": "Learn how to use Schema.org markup for better SEO.",
  "mainEntityOfPage": {
    "@type": "WebPage",
    "@id": "https://example.com/guides/schema-org"
  }
}
```

### Person

**Description:** Individual person

**Key Properties:**
- `name`: Full name
- `email`: Email address
- `url`: Personal website
- `image`: Photo
- `jobTitle`: Job title
- `worksFor`: Organization (Organization type)
- `sameAs`: Social media profiles (array of URLs)

**Example:**

```json
{
  "@context": "https://schema.org",
  "@type": "Person",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "url": "https://janedoe.dev",
  "image": "https://example.com/jane.jpg",
  "jobTitle": "Senior Software Engineer",
  "worksFor": {
    "@type": "Organization",
    "name": "Example Company",
    "url": "https://example.com"
  },
  "sameAs": [
    "https://twitter.com/janedoe",
    "https://github.com/janedoe",
    "https://linkedin.com/in/janedoe"
  ],
  "alumniOf": {
    "@type": "EducationalOrganization",
    "name": "MIT"
  },
  "knowsAbout": ["Web Development", "JavaScript", "C#"]
}
```

### Organization

**Description:** Company, non-profit, institution, etc.

**Key Properties:**
- `name`: Organization name
- `url`: Website URL
- `logo`: Logo image
- `description`: Description
- `contactPoint`: Contact information
- `sameAs`: Social media profiles
- `address`: Physical address

**Subtypes:**
- `Corporation`
- `EducationalOrganization`
- `GovernmentOrganization`
- `LocalBusiness`
- `NGO`

**Example:**

```json
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "Example Company",
  "url": "https://example.com",
  "logo": "https://example.com/logo.png",
  "description": "Leading provider of web development tools",
  "foundingDate": "2020-01-15",
  "founders": [
    {
      "@type": "Person",
      "name": "Jane Doe"
    }
  ],
  "address": {
    "@type": "PostalAddress",
    "streetAddress": "123 Main Street",
    "addressLocality": "San Francisco",
    "addressRegion": "CA",
    "postalCode": "94105",
    "addressCountry": "US"
  },
  "contactPoint": {
    "@type": "ContactPoint",
    "telephone": "+1-555-123-4567",
    "contactType": "Customer Service",
    "email": "support@example.com",
    "areaServed": "US",
    "availableLanguage": ["English", "Spanish"]
  },
  "sameAs": [
    "https://twitter.com/example",
    "https://facebook.com/example",
    "https://linkedin.com/company/example",
    "https://github.com/example"
  ]
}
```

### Product

**Description:** Physical or digital product

**Required Properties:**
- `name`: Product name
- `image`: Product image
- `offers`: Offer (price, availability)

**Recommended Properties:**
- `description`: Product description
- `brand`: Brand (Brand or Organization)
- `sku`: Stock keeping unit
- `gtin`: Global trade item number
- `aggregateRating`: Customer ratings
- `review`: Customer reviews

**Example:**

```json
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Wireless Headphones Pro",
  "image": "https://example.com/products/headphones.jpg",
  "description": "Premium noise-cancelling wireless headphones",
  "sku": "WHP-2026",
  "mpn": "925872",
  "brand": {
    "@type": "Brand",
    "name": "AudioTech"
  },
  "offers": {
    "@type": "Offer",
    "url": "https://example.com/products/headphones",
    "priceCurrency": "USD",
    "price": "299.99",
    "priceValidUntil": "2026-12-31",
    "availability": "https://schema.org/InStock",
    "itemCondition": "https://schema.org/NewCondition",
    "seller": {
      "@type": "Organization",
      "name": "Example Store"
    }
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.7",
    "reviewCount": "342",
    "bestRating": "5",
    "worstRating": "1"
  },
  "review": {
    "@type": "Review",
    "reviewRating": {
      "@type": "Rating",
      "ratingValue": "5"
    },
    "author": {
      "@type": "Person",
      "name": "John Smith"
    },
    "reviewBody": "Amazing sound quality and comfort!"
  }
}
```

### LocalBusiness

**Description:** Physical business location

**Key Properties:**
- `name`: Business name
- `image`: Business photo
- `address`: Physical address
- `telephone`: Phone number
- `openingHoursSpecification`: Business hours
- `geo`: Geographic coordinates
- `priceRange`: Price range (e.g., "$$")

**Subtypes:**
- `Restaurant`
- `Store`
- `FoodEstablishment`
- `AutomotiveBusiness`

**Example:**

```json
{
  "@context": "https://schema.org",
  "@type": "Restaurant",
  "name": "The Example Cafe",
  "image": "https://example.com/cafe.jpg",
  "url": "https://example.com",
  "telephone": "+1-555-987-6543",
  "address": {
    "@type": "PostalAddress",
    "streetAddress": "456 Cafe Street",
    "addressLocality": "San Francisco",
    "addressRegion": "CA",
    "postalCode": "94102",
    "addressCountry": "US"
  },
  "geo": {
    "@type": "GeoCoordinates",
    "latitude": "37.7749",
    "longitude": "-122.4194"
  },
  "openingHoursSpecification": [
    {
      "@type": "OpeningHoursSpecification",
      "dayOfWeek": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"],
      "opens": "08:00",
      "closes": "18:00"
    },
    {
      "@type": "OpeningHoursSpecification",
      "dayOfWeek": ["Saturday", "Sunday"],
      "opens": "09:00",
      "closes": "17:00"
    }
  ],
  "servesCuisine": "American",
  "priceRange": "$$",
  "acceptsReservations": "True"
}
```

### Event

**Description:** Scheduled event (conference, concert, webinar, etc.)

**Required Properties:**
- `name`: Event name
- `startDate`: Start date and time
- `location`: Event location (Place or VirtualLocation)

**Recommended Properties:**
- `endDate`: End date and time
- `image`: Event image
- `description`: Event description
- `offers`: Ticket pricing
- `performer`: Speakers/performers
- `organizer`: Event organizer

**Example:**

```json
{
  "@context": "https://schema.org",
  "@type": "Event",
  "name": "Web Development Summit 2026",
  "description": "Annual conference for web developers",
  "image": "https://example.com/events/summit-2026.jpg",
  "startDate": "2026-06-15T09:00:00-07:00",
  "endDate": "2026-06-17T18:00:00-07:00",
  "eventStatus": "https://schema.org/EventScheduled",
  "eventAttendanceMode": "https://schema.org/OfflineEventAttendanceMode",
  "location": {
    "@type": "Place",
    "name": "Moscone Center",
    "address": {
      "@type": "PostalAddress",
      "streetAddress": "747 Howard St",
      "addressLocality": "San Francisco",
      "addressRegion": "CA",
      "postalCode": "94103",
      "addressCountry": "US"
    }
  },
  "offers": {
    "@type": "Offer",
    "url": "https://example.com/events/summit-2026/register",
    "price": "599",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock",
    "validFrom": "2026-01-01"
  },
  "performer": [
    {
      "@type": "Person",
      "name": "Jane Doe"
    },
    {
      "@type": "Person",
      "name": "John Smith"
    }
  ],
  "organizer": {
    "@type": "Organization",
    "name": "Tech Events Inc",
    "url": "https://techevents.example"
  }
}
```

### Recipe

**Description:** Cooking recipe

**Required Properties:**
- `name`: Recipe name
- `image`: Recipe image
- `recipeIngredient`: List of ingredients

**Recommended Properties:**
- `prepTime`: Preparation time (ISO 8601 duration)
- `cookTime`: Cooking time
- `totalTime`: Total time
- `recipeYield`: Number of servings
- `recipeInstructions`: Step-by-step instructions
- `nutrition`: Nutritional information

**Example:**

```json
{
  "@context": "https://schema.org",
  "@type": "Recipe",
  "name": "Classic Chocolate Chip Cookies",
  "image": "https://example.com/recipes/cookies.jpg",
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  },
  "datePublished": "2026-02-13",
  "description": "Delicious homemade chocolate chip cookies",
  "prepTime": "PT15M",
  "cookTime": "PT12M",
  "totalTime": "PT27M",
  "recipeYield": "24 cookies",
  "recipeCategory": "Dessert",
  "recipeCuisine": "American",
  "keywords": "cookies, chocolate chip, dessert",
  "nutrition": {
    "@type": "NutritionInformation",
    "calories": "150 calories",
    "carbohydrateContent": "20 g",
    "proteinContent": "2 g",
    "fatContent": "7 g"
  },
  "recipeIngredient": [
    "2 1/4 cups all-purpose flour",
    "1 tsp baking soda",
    "1 cup butter, softened",
    "3/4 cup granulated sugar",
    "2 cups chocolate chips"
  ],
  "recipeInstructions": [
    {
      "@type": "HowToStep",
      "text": "Preheat oven to 375°F (190°C)."
    },
    {
      "@type": "HowToStep",
      "text": "Mix flour and baking soda in a bowl."
    },
    {
      "@type": "HowToStep",
      "text": "Beat butter and sugars until creamy."
    },
    {
      "@type": "HowToStep",
      "text": "Add chocolate chips and mix well."
    },
    {
      "@type": "HowToStep",
      "text": "Bake for 10-12 minutes until golden brown."
    }
  ],
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.8",
    "reviewCount": "156"
  }
}
```

### VideoObject

**Description:** Video content

**Required Properties:**
- `name`: Video title
- `description`: Video description
- `thumbnailUrl`: Thumbnail image
- `uploadDate`: Upload date

**Recommended Properties:**
- `duration`: Video duration (ISO 8601)
- `contentUrl`: Direct video file URL
- `embedUrl`: Embed player URL
- `interactionStatistic`: View count

**Example:**

```json
{
  "@context": "https://schema.org",
  "@type": "VideoObject",
  "name": "Introduction to Schema.org",
  "description": "Learn the basics of Schema.org structured data",
  "thumbnailUrl": "https://example.com/video-thumb.jpg",
  "uploadDate": "2026-02-13T10:00:00Z",
  "duration": "PT15M32S",
  "contentUrl": "https://example.com/videos/schema-intro.mp4",
  "embedUrl": "https://example.com/embed/video-123",
  "width": 1920,
  "height": 1080,
  "interactionStatistic": {
    "@type": "InteractionCounter",
    "interactionType": "https://schema.org/WatchAction",
    "userInteractionCount": 12847
  },
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  }
}
```

## Specialized Types

### FAQPage

**Description:** Frequently asked questions page

**Structure:** List of Question/Answer pairs

**Example:**

```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "What is Schema.org?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Schema.org is a collaborative vocabulary for structured data markup on web pages."
      }
    },
    {
      "@type": "Question",
      "name": "Why use Schema.org?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Schema.org helps search engines understand your content and can enable rich results."
      }
    }
  ]
}
```

### HowTo

**Description:** Step-by-step instructions

**Required Properties:**
- `name`: Title of how-to
- `step`: Array of HowToStep objects

**Example:**

```json
{
  "@context": "https://schema.org",
  "@type": "HowTo",
  "name": "How to Add Schema.org Markup",
  "description": "Step-by-step guide to implementing structured data",
  "image": "https://example.com/how-to-schema.jpg",
  "totalTime": "PT30M",
  "estimatedCost": {
    "@type": "MonetaryAmount",
    "currency": "USD",
    "value": "0"
  },
  "tool": {
    "@type": "HowToTool",
    "name": "Text editor"
  },
  "step": [
    {
      "@type": "HowToStep",
      "name": "Choose Schema type",
      "text": "Select the appropriate Schema.org type for your content",
      "url": "https://example.com/guide#step1",
      "image": "https://example.com/step1.jpg"
    },
    {
      "@type": "HowToStep",
      "name": "Add JSON-LD script",
      "text": "Insert a script tag with type='application/ld+json'",
      "url": "https://example.com/guide#step2"
    }
  ]
}
```

### BreadcrumbList

**Description:** Breadcrumb navigation

**Structure:** Ordered list of pages

**Example:**

```json
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
      "name": "Guides",
      "item": "https://example.com/guides"
    },
    {
      "@type": "ListItem",
      "position": 3,
      "name": "Schema.org",
      "item": "https://example.com/guides/schema-org"
    }
  ]
}
```

## Property Types

### Text Properties

```json
"headline": "Article Title",
"description": "Article description text",
"name": "Product Name"
```

### Date/DateTime Properties

**Format:** ISO 8601

```json
"datePublished": "2026-02-13",
"dateModified": "2026-02-13T15:30:00Z",
"startDate": "2026-06-15T09:00:00-07:00"
```

### Duration Properties

**Format:** ISO 8601 duration

```json
"duration": "PT15M32S",  // 15 minutes, 32 seconds
"prepTime": "PT30M",     // 30 minutes
"cookTime": "PT1H15M"    // 1 hour, 15 minutes
```

### URL Properties

**Format:** Absolute URL (https://)

```json
"url": "https://example.com",
"image": "https://example.com/image.jpg",
"contentUrl": "https://example.com/video.mp4"
```

### Nested Objects

```json
"author": {
  "@type": "Person",
  "name": "Jane Doe",
  "url": "https://example.com/jane"
},
"address": {
  "@type": "PostalAddress",
  "streetAddress": "123 Main St",
  "addressLocality": "San Francisco"
}
```

### Arrays

```json
"recipeIngredient": [
  "2 cups flour",
  "1 cup sugar",
  "3 eggs"
],
"sameAs": [
  "https://twitter.com/example",
  "https://facebook.com/example"
]
```

## Hugo Implementation Patterns

### Conditional Schema Types

```go-html-template
{{/* layouts/partials/schema.html */}}

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  {{- if eq .Type "post" }}
  "@type": "BlogPosting",
  {{- else if eq .Type "product" }}
  "@type": "Product",
  {{- else if eq .Type "event" }}
  "@type": "Event",
  {{- else }}
  "@type": "WebPage",
  {{- end }}
  "name": {{ .Title | jsonify }},
  "url": {{ .Permalink | jsonify }}
}
</script>
```

### Front Matter Driven

```yaml
---
title: "Product Name"
type: product
schema:
  type: Product
  sku: "PRD-123"
  price: 99.99
  currency: USD
  availability: InStock
---
```

```go-html-template
{{- if .Params.schema }}
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": {{ .Params.schema.type | jsonify }},
  "name": {{ .Title | jsonify }},
  {{- with .Params.schema.sku }}
  "sku": {{ . | jsonify }},
  {{- end }}
  {{- with .Params.schema.price }}
  "offers": {
    "@type": "Offer",
    "price": {{ . | jsonify }},
    "priceCurrency": {{ $.Params.schema.currency | default "USD" | jsonify }}
  }
  {{- end }}
}
</script>
{{- end }}
```

## Best Practices

**Type Selection:**
- Choose most specific type (BlogPosting vs Article)
- Use subtypes when available
- Don't mix unrelated types

**Properties:**
- Include all required properties
- Add recommended properties when possible
- Match schema data to visible content

**Values:**
- Use absolute URLs (not relative)
- Follow ISO 8601 for dates/times
- Provide accurate, current information

**Testing:**
- Google Rich Results Test
- Schema.org Validator
- Google Search Console monitoring

## Common Patterns

### Multiple Types on One Page

```json
[
  {
    "@context": "https://schema.org",
    "@type": "Article",
    ...
  },
  {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    ...
  }
]
```

### Referenced Entities

```json
{
  "@context": "https://schema.org",
  "@type": "Article",
  "author": {
    "@id": "https://example.com/authors/jane-doe#person"
  }
}
```

```json
{
  "@context": "https://schema.org",
  "@type": "Person",
  "@id": "https://example.com/authors/jane-doe#person",
  "name": "Jane Doe"
}
```

## Guidelines

**Required:**
- `@context`: Always "https://schema.org"
- `@type`: Choose appropriate type
- Type-required properties (varies by type)

**Recommended:**
- Use most specific type available
- Include as many relevant properties as possible
- Nest objects for complex relationships
- Provide absolute URLs

**Testing:**
- Google Rich Results Test
- Schema.org Validator
- Monitor Search Console

## Benefits

Semantic. Machines understand content meaning.

Discoverable. Better search engine comprehension.

Enhanced. Rich results in search (snippets, panels).

Standardized. Universal vocabulary across platforms.

## Related

- [structured-data-json-ld.md](./structured-data-json-ld.md) - JSON-LD implementation
- [open-graph-protocol.md](./open-graph-protocol.md) - Social media metadata
- [meta-tags-seo.md](./meta-tags-seo.md) - HTML meta tags
- [sitemap-xml-advanced.md](./sitemap-xml-advanced.md) - XML sitemaps
