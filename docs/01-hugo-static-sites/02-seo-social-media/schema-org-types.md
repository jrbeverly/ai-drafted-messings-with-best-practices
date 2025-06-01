# Schema.org Types Reference

Schema.org type catalog. Rich results types. Structured data reference. JSON-LD examples. SEO markup guide.

## Principle

Choose the correct Schema.org type for your content. Use specific types over generic ones. Implement required properties for rich results. Maximize search engine understanding and rich snippet eligibility.

## Schema.org Hierarchy

**Schema.org uses hierarchical inheritance:**

```
Thing (root type)
├── CreativeWork
│   ├── Article
│   │   ├── BlogPosting
│   │   ├── NewsArticle
│   │   ├── Report
│   │   └── TechArticle
│   ├── Book
│   ├── Course
│   ├── Movie
│   ├── SoftwareApplication
│   └── WebPage
│       ├── FAQPage
│       ├── QAPage
│       └── ProfilePage
├── Person
├── Organization
│   ├── Corporation
│   ├── EducationalOrganization
│   ├── LocalBusiness
│   └── NGO
├── Event
│   ├── BusinessEvent
│   ├── EducationEvent
│   └── SocialEvent
├── Product
│   └── SoftwareApplication
├── Place
│   ├── LocalBusiness
│   └── Residence
└── Intangible
    ├── ItemList
    ├── BreadcrumbList
    └── Rating
```

**Principle:** Use the most specific type that fits your content.

**Example:**
- ❌ Don't use `Article` for blog posts
- ✅ Use `BlogPosting` (more specific)

## Article Types

### Article (Generic)

**Use for:** General articles not fitting other subtypes

**Required:**
- headline
- image
- datePublished
- author

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "The Future of Static Sites",
  "image": "https://example.com/images/article.jpg",
  "datePublished": "2026-02-13T10:00:00Z",
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  }
}
</script>
```

### BlogPosting

**Use for:** Blog posts

**Required:** Same as Article

**Additional properties:**
- blogName (optional)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BlogPosting",
  "headline": "10 Hugo Performance Tips",
  "image": "https://example.com/images/blog-post.jpg",
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
      "url": "https://example.com/logo.png"
    }
  },
  "description": "Learn 10 proven techniques to optimize Hugo sites",
  "mainEntityOfPage": "https://example.com/blog/hugo-tips/"
}
</script>
```

### NewsArticle

**Use for:** News articles and journalism

**Required:** Same as Article

**Additional properties:**
- dateline (location)
- printEdition
- printPage
- printSection

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "NewsArticle",
  "headline": "Hugo 0.120.0 Released with Major Performance Improvements",
  "image": "https://example.com/images/news.jpg",
  "datePublished": "2026-02-13T08:00:00Z",
  "dateModified": "2026-02-13T12:00:00Z",
  "author": {
    "@type": "Person",
    "name": "Tech Reporter"
  },
  "publisher": {
    "@type": "Organization",
    "name": "Hugo News",
    "logo": {
      "@type": "ImageObject",
      "url": "https://example.com/logo.png"
    }
  },
  "dateline": "San Francisco, CA"
}
</script>
```

### TechArticle

**Use for:** Technical articles and how-to guides

**Required:** Same as Article

**Additional properties:**
- dependencies
- proficiencyLevel
- skillLevel

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Complete Guide to Hugo Modules",
  "image": "https://example.com/images/tech-guide.jpg",
  "datePublished": "2026-02-13T10:00:00Z",
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  },
  "publisher": {
    "@type": "Organization",
    "name": "Hugo Docs"
  },
  "proficiencyLevel": "Intermediate",
  "dependencies": "Hugo v0.100+, Go modules knowledge"
}
</script>
```

### Report

**Use for:** Research reports, whitepapers, studies

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Report",
  "headline": "Static Site Performance Benchmark Report 2026",
  "image": "https://example.com/images/report.jpg",
  "datePublished": "2026-02-13T10:00:00Z",
  "author": {
    "@type": "Organization",
    "name": "Static Site Research Group"
  },
  "publisher": {
    "@type": "Organization",
    "name": "Web Performance Institute"
  },
  "about": "Comparative analysis of static site generator performance"
}
</script>
```

## Person

**Use for:** Author pages, bios, profiles

**Required:**
- name

**Recommended:**
- url
- image
- jobTitle
- sameAs (social profiles)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Person",
  "name": "Jane Doe",
  "url": "https://example.com/authors/jane-doe/",
  "image": "https://example.com/images/jane-doe.jpg",
  "jobTitle": "Senior Web Developer",
  "worksFor": {
    "@type": "Organization",
    "name": "Hugo Best Practices"
  },
  "sameAs": [
    "https://twitter.com/janedoe",
    "https://github.com/janedoe",
    "https://linkedin.com/in/janedoe"
  ],
  "email": "jane@example.com",
  "description": "Web developer specializing in Hugo and static sites",
  "knowsAbout": ["Hugo", "Static Sites", "Web Performance", "JAMstack"]
}
</script>
```

## Organization

**Use for:** Company pages, organizations

**Required:**
- name
- url

**Recommended:**
- logo
- contactPoint
- sameAs

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "Hugo Best Practices",
  "url": "https://example.com",
  "logo": "https://example.com/images/logo.png",
  "description": "Resources and tutorials for building fast static sites",
  "foundingDate": "2020-01-15",
  "founders": [
    {
      "@type": "Person",
      "name": "Jane Doe"
    }
  ],
  "address": {
    "@type": "PostalAddress",
    "streetAddress": "123 Main St",
    "addressLocality": "San Francisco",
    "addressRegion": "CA",
    "postalCode": "94102",
    "addressCountry": "US"
  },
  "contactPoint": {
    "@type": "ContactPoint",
    "contactType": "Customer Service",
    "email": "hello@example.com",
    "url": "https://example.com/contact/",
    "availableLanguage": ["English"]
  },
  "sameAs": [
    "https://twitter.com/hugobest",
    "https://github.com/hugobest",
    "https://facebook.com/hugobest"
  ]
}
</script>
```

## WebSite

**Use for:** Homepage, site-wide information

**Required:**
- name
- url

**Recommended:**
- potentialAction (SearchAction)

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
  },
  "inLanguage": "en-US"
}
</script>
```

## WebPage Types

### WebPage (Generic)

**Use for:** Standard pages

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "WebPage",
  "name": "About Us",
  "url": "https://example.com/about/",
  "description": "Learn about our mission and team",
  "inLanguage": "en-US",
  "isPartOf": {
    "@type": "WebSite",
    "url": "https://example.com"
  }
}
</script>
```

### FAQPage

**Use for:** FAQ pages (enables FAQ rich results)

**Required:**
- mainEntity (array of Questions)

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
        "text": "Hugo is a fast static site generator written in Go. It builds websites in milliseconds and is popular for blogs, documentation, and marketing sites."
      }
    },
    {
      "@type": "Question",
      "name": "Is Hugo free?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Yes, Hugo is open source and completely free to use under the Apache 2.0 license."
      }
    },
    {
      "@type": "Question",
      "name": "What languages does Hugo support?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Hugo supports multilingual sites and can build content in any language. It has built-in i18n support for translations."
      }
    }
  ]
}
</script>
```

### QAPage

**Use for:** Question/answer pages (like Stack Overflow style)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "QAPage",
  "mainEntity": {
    "@type": "Question",
    "name": "How do I optimize Hugo build times?",
    "text": "My Hugo site takes 30 seconds to build. How can I make it faster?",
    "answerCount": 2,
    "upvoteCount": 15,
    "dateCreated": "2026-02-13T10:00:00Z",
    "author": {
      "@type": "Person",
      "name": "John Smith"
    },
    "acceptedAnswer": {
      "@type": "Answer",
      "text": "Enable caching with --gc flag, use partial caching, and optimize image processing...",
      "dateCreated": "2026-02-13T11:00:00Z",
      "upvoteCount": 20,
      "url": "https://example.com/questions/123#answer-456",
      "author": {
        "@type": "Person",
        "name": "Jane Doe"
      }
    },
    "suggestedAnswer": [
      {
        "@type": "Answer",
        "text": "Another approach is to use incremental builds...",
        "dateCreated": "2026-02-13T12:00:00Z",
        "upvoteCount": 5,
        "author": {
          "@type": "Person",
          "name": "Bob Johnson"
        }
      }
    ]
  }
}
</script>
```

### ProfilePage

**Use for:** User profile pages

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "ProfilePage",
  "mainEntity": {
    "@type": "Person",
    "name": "Jane Doe",
    "image": "https://example.com/images/jane.jpg"
  },
  "url": "https://example.com/profile/janedoe/",
  "dateCreated": "2020-01-15",
  "dateModified": "2026-02-13"
}
</script>
```

## BreadcrumbList

**Use for:** Breadcrumb navigation

**Required:**
- itemListElement (array)

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
      "name": "Hugo Tips",
      "item": "https://example.com/blog/hugo-tips/"
    }
  ]
}
</script>
```

## ItemList

**Use for:** Lists of items (blog posts, products, etc.)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "ItemList",
  "numberOfItems": 10,
  "itemListElement": [
    {
      "@type": "ListItem",
      "position": 1,
      "url": "https://example.com/blog/post1/",
      "name": "First Blog Post"
    },
    {
      "@type": "ListItem",
      "position": 2,
      "url": "https://example.com/blog/post2/",
      "name": "Second Blog Post"
    }
  ]
}
</script>
```

## HowTo

**Use for:** Step-by-step tutorials and how-to guides

**Required:**
- name
- step (array of HowToSteps)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "HowTo",
  "name": "How to Install Hugo",
  "description": "Step-by-step guide to installing Hugo on your system",
  "image": "https://example.com/images/hugo-install.jpg",
  "totalTime": "PT10M",
  "estimatedCost": {
    "@type": "MonetaryAmount",
    "currency": "USD",
    "value": "0"
  },
  "tool": [
    {
      "@type": "HowToTool",
      "name": "Terminal or Command Prompt"
    }
  ],
  "step": [
    {
      "@type": "HowToStep",
      "name": "Download Hugo",
      "text": "Visit the Hugo releases page and download the appropriate binary for your operating system",
      "url": "https://example.com/install#step1",
      "image": "https://example.com/images/step1.jpg"
    },
    {
      "@type": "HowToStep",
      "name": "Install Hugo",
      "text": "Extract the downloaded archive and move the hugo executable to your PATH",
      "url": "https://example.com/install#step2",
      "image": "https://example.com/images/step2.jpg"
    },
    {
      "@type": "HowToStep",
      "name": "Verify Installation",
      "text": "Open a terminal and run 'hugo version' to verify the installation was successful",
      "url": "https://example.com/install#step3",
      "image": "https://example.com/images/step3.jpg"
    }
  ]
}
</script>
```

## Course

**Use for:** Online courses and educational content

**Required:**
- name
- description

**Recommended:**
- provider
- hasCourseInstance

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Course",
  "name": "Complete Hugo Masterclass",
  "description": "Learn to build production-ready static sites with Hugo from scratch",
  "provider": {
    "@type": "Organization",
    "name": "Hugo Academy",
    "sameAs": "https://hugoacademy.com"
  },
  "image": "https://example.com/images/course.jpg",
  "hasCourseInstance": {
    "@type": "CourseInstance",
    "courseMode": "online",
    "courseWorkload": "PT10H",
    "instructor": {
      "@type": "Person",
      "name": "Jane Doe"
    }
  },
  "offers": {
    "@type": "Offer",
    "price": "99.00",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock"
  }
}
</script>
```

## Event

**Use for:** Events, conferences, webinars

**Required:**
- name
- startDate
- location (Place or VirtualLocation)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Event",
  "name": "HugoCon 2026",
  "description": "Annual Hugo static site generator conference",
  "image": "https://example.com/images/hugocon.jpg",
  "startDate": "2026-05-15T09:00:00-07:00",
  "endDate": "2026-05-17T17:00:00-07:00",
  "eventStatus": "https://schema.org/EventScheduled",
  "eventAttendanceMode": "https://schema.org/OfflineEventAttendanceMode",
  "location": {
    "@type": "Place",
    "name": "San Francisco Convention Center",
    "address": {
      "@type": "PostalAddress",
      "streetAddress": "747 Howard St",
      "addressLocality": "San Francisco",
      "addressRegion": "CA",
      "postalCode": "94103",
      "addressCountry": "US"
    }
  },
  "organizer": {
    "@type": "Organization",
    "name": "Hugo Foundation",
    "url": "https://hugofoundation.org"
  },
  "offers": {
    "@type": "Offer",
    "price": "299.00",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock",
    "url": "https://example.com/tickets",
    "validFrom": "2026-01-01"
  }
}
</script>
```

### Virtual Event

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Event",
  "name": "Hugo Webinar: Performance Optimization",
  "description": "Learn advanced Hugo performance techniques",
  "image": "https://example.com/images/webinar.jpg",
  "startDate": "2026-03-10T14:00:00Z",
  "endDate": "2026-03-10T15:30:00Z",
  "eventStatus": "https://schema.org/EventScheduled",
  "eventAttendanceMode": "https://schema.org/OnlineEventAttendanceMode",
  "location": {
    "@type": "VirtualLocation",
    "url": "https://example.com/webinar/join"
  },
  "organizer": {
    "@type": "Organization",
    "name": "Hugo Academy"
  },
  "offers": {
    "@type": "Offer",
    "price": "0",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock",
    "url": "https://example.com/webinar/register"
  }
}
</script>
```

## Product

**Use for:** Products, software, themes

**Required:**
- name
- image

**Recommended:**
- offers (price)
- aggregateRating (reviews)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Hugo Pro Theme",
  "image": "https://example.com/images/theme.jpg",
  "description": "Professional Hugo theme with 50+ components and dark mode",
  "brand": {
    "@type": "Organization",
    "name": "Hugo Themes"
  },
  "offers": {
    "@type": "Offer",
    "price": "49.00",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock",
    "url": "https://example.com/themes/hugo-pro/",
    "priceValidUntil": "2026-12-31"
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.8",
    "reviewCount": "127",
    "bestRating": "5",
    "worstRating": "1"
  },
  "review": [
    {
      "@type": "Review",
      "author": {
        "@type": "Person",
        "name": "John Smith"
      },
      "datePublished": "2026-02-10",
      "reviewBody": "Excellent theme with great documentation. Easy to customize.",
      "reviewRating": {
        "@type": "Rating",
        "ratingValue": "5",
        "bestRating": "5"
      }
    }
  ]
}
</script>
```

## SoftwareApplication

**Use for:** Software, apps, tools

**Required:**
- name
- applicationCategory

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "SoftwareApplication",
  "name": "Hugo CLI",
  "applicationCategory": "DeveloperApplication",
  "operatingSystem": ["Windows", "macOS", "Linux"],
  "offers": {
    "@type": "Offer",
    "price": "0",
    "priceCurrency": "USD"
  },
  "softwareVersion": "0.120.0",
  "fileSize": "15MB",
  "downloadUrl": "https://github.com/gohugoio/hugo/releases",
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.9",
    "ratingCount": "5000"
  }
}
</script>
```

## Book

**Use for:** Books, ebooks, documentation

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Book",
  "name": "Mastering Hugo: The Definitive Guide",
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  },
  "isbn": "978-0-123456-78-9",
  "bookFormat": "https://schema.org/EBook",
  "datePublished": "2026-02-13",
  "publisher": {
    "@type": "Organization",
    "name": "Tech Publishing"
  },
  "numberOfPages": 450,
  "inLanguage": "en-US",
  "offers": {
    "@type": "Offer",
    "price": "29.99",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock"
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.7",
    "reviewCount": "89"
  }
}
</script>
```

## VideoObject

**Use for:** Video content

**Required:**
- name
- description
- thumbnailUrl
- uploadDate

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "VideoObject",
  "name": "Hugo Tutorial: Getting Started",
  "description": "Learn Hugo basics in 15 minutes",
  "thumbnailUrl": "https://example.com/images/video-thumb.jpg",
  "uploadDate": "2026-02-13T10:00:00Z",
  "duration": "PT15M30S",
  "contentUrl": "https://example.com/videos/hugo-tutorial.mp4",
  "embedUrl": "https://example.com/embed/hugo-tutorial",
  "interactionStatistic": {
    "@type": "InteractionCounter",
    "interactionType": "https://schema.org/WatchAction",
    "userInteractionCount": 5000
  }
}
</script>
```

## LocalBusiness

**Use for:** Local businesses with physical location

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "LocalBusiness",
  "name": "Hugo Web Design Studio",
  "image": "https://example.com/images/studio.jpg",
  "address": {
    "@type": "PostalAddress",
    "streetAddress": "123 Main St",
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
  "telephone": "+1-415-555-0123",
  "openingHoursSpecification": [
    {
      "@type": "OpeningHoursSpecification",
      "dayOfWeek": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"],
      "opens": "09:00",
      "closes": "17:00"
    }
  ],
  "priceRange": "$$",
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.8",
    "reviewCount": "45"
  }
}
</script>
```

## Recipe

**Use for:** Cooking recipes (can adapt for technical "recipes")

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Recipe",
  "name": "Perfect Hugo Site Setup",
  "image": "https://example.com/images/recipe.jpg",
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  },
  "datePublished": "2026-02-13",
  "description": "Step-by-step recipe for setting up a production Hugo site",
  "prepTime": "PT30M",
  "cookTime": "PT1H",
  "totalTime": "PT1H30M",
  "recipeYield": "1 website",
  "recipeIngredient": [
    "Hugo v0.120+",
    "Git",
    "Node.js (optional)",
    "Domain name"
  ],
  "recipeInstructions": [
    {
      "@type": "HowToStep",
      "text": "Install Hugo on your system"
    },
    {
      "@type": "HowToStep",
      "text": "Create new site with 'hugo new site mysite'"
    },
    {
      "@type": "HowToStep",
      "text": "Choose and install a theme"
    }
  ]
}
</script>
```

## JobPosting

**Use for:** Job listings

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "JobPosting",
  "title": "Senior Hugo Developer",
  "description": "We're looking for an experienced Hugo developer to join our team",
  "datePosted": "2026-02-13",
  "validThrough": "2026-03-31",
  "employmentType": "FULL_TIME",
  "hiringOrganization": {
    "@type": "Organization",
    "name": "Hugo Best Practices",
    "sameAs": "https://example.com"
  },
  "jobLocation": {
    "@type": "Place",
    "address": {
      "@type": "PostalAddress",
      "addressLocality": "San Francisco",
      "addressRegion": "CA",
      "addressCountry": "US"
    }
  },
  "baseSalary": {
    "@type": "MonetaryAmount",
    "currency": "USD",
    "value": {
      "@type": "QuantitativeValue",
      "minValue": 100000,
      "maxValue": 150000,
      "unitText": "YEAR"
    }
  }
}
</script>
```

## Type Selection Guide

### Decision Tree

```
What type of content?

├─ Written content?
│  ├─ Blog post? → BlogPosting
│  ├─ News? → NewsArticle
│  ├─ Technical guide? → TechArticle
│  ├─ Research? → Report
│  └─ General article? → Article
│
├─ Person/organization?
│  ├─ Individual? → Person
│  ├─ Company? → Organization
│  └─ Local business? → LocalBusiness
│
├─ List/navigation?
│  ├─ Breadcrumbs? → BreadcrumbList
│  └─ Item list? → ItemList
│
├─ Page type?
│  ├─ FAQ page? → FAQPage
│  ├─ Q&A? → QAPage
│  ├─ Profile? → ProfilePage
│  ├─ Homepage? → WebSite
│  └─ General page? → WebPage
│
├─ Tutorial/guide?
│  ├─ Step-by-step? → HowTo
│  └─ Course? → Course
│
├─ Product/service?
│  ├─ Physical/digital product? → Product
│  ├─ Software? → SoftwareApplication
│  └─ Book? → Book
│
├─ Event?
│  ├─ In-person? → Event (OfflineEventAttendanceMode)
│  └─ Virtual? → Event (OnlineEventAttendanceMode)
│
└─ Media?
   ├─ Video? → VideoObject
   └─ Audio? → AudioObject
```

## Content Type Mapping

**Common Hugo content types → Schema.org types:**

| Hugo Content Type | Schema.org Type | Notes |
|-------------------|-----------------|-------|
| blog/ | BlogPosting | Blog posts |
| news/ | NewsArticle | News articles |
| docs/ | TechArticle | Documentation |
| tutorial/ | HowTo | Step-by-step guides |
| course/ | Course | Educational content |
| authors/ | Person | Author profiles |
| about/ | Organization | Company info |
| products/ | Product | Products/themes |
| events/ | Event | Events/webinars |
| faq/ | FAQPage | FAQ pages |
| videos/ | VideoObject | Video content |

## Hugo Implementation

### Dynamic Type Selection

**layouts/partials/structured-data/auto-type.html:**

```go-html-template
{{ $schemaType := "WebPage" }}

{{ if .IsHome }}
  {{ $schemaType = "WebSite" }}
{{ else if eq .Type "blog" }}
  {{ $schemaType = "BlogPosting" }}
{{ else if eq .Type "news" }}
  {{ $schemaType = "NewsArticle" }}
{{ else if eq .Type "tutorial" }}
  {{ $schemaType = "TechArticle" }}
{{ else if eq .Type "author" }}
  {{ $schemaType = "Person" }}
{{ else if eq .Type "product" }}
  {{ $schemaType = "Product" }}
{{ else if eq .Type "event" }}
  {{ $schemaType = "Event" }}
{{ else if .Params.schema_type }}
  {{ $schemaType = .Params.schema_type }}
{{ end }}

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": {{ $schemaType }},
  ...
}
</script>
```

### Front Matter Override

```yaml
---
title: "My FAQ Page"
schema_type: FAQPage  # Override automatic detection
---
```

## Best Practices

### Type Selection

**✅ DO:**
- Use most specific type that fits
- BlogPosting for blog posts (not Article)
- TechArticle for technical guides
- FAQPage for FAQ pages
- HowTo for step-by-step tutorials

**❌ DON'T:**
- Use generic types when specific ones exist
- Mix types inappropriately
- Use Article for everything

### Required Fields

**✅ DO:**
- Include all required fields for type
- Check Google Rich Results documentation
- Validate with Google Rich Results Test
- Provide recommended fields

**❌ DON'T:**
- Omit required fields
- Use placeholder data
- Skip validation

### Images

**✅ DO:**
- Use high-quality images (1200px+ wide)
- Provide multiple aspect ratios
- Use absolute HTTPS URLs
- Specify width/height for logos

**❌ DON'T:**
- Use small images (< 1200px wide)
- Use relative URLs
- Forget image alt text (separate meta tag)

## Guidelines

### Essential

**Minimum for any type:**
- @context (https://schema.org)
- @type (specific type)
- All required fields for that type
- Absolute HTTPS URLs
- ISO 8601 dates

### Recommended

**For better rich results:**
- Multiple image sizes
- Complete author/publisher info
- dateModified (if applicable)
- description
- All recommended fields

### Advanced

**For maximum SEO benefit:**
- Multiple schemas with @graph
- Nested entities
- Review/rating data
- FAQ/HowTo where applicable
- Complete Organization data

## Related

- [structured-data-schema-org.md](./structured-data-schema-org.md) - Schema.org implementation basics
- [open-graph-meta-tags.md](./open-graph-meta-tags.md) - Social media metadata
- [meta-descriptions-titles.md](./meta-descriptions-titles.md) - Title and description optimization
- [sitemaps-robots-txt.md](./sitemaps-robots-txt.md) - Sitemap generation
