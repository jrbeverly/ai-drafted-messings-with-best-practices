# Pinterest Rich Pins

Pinterest Rich Pins. Pin optimization. Visual content meta tags. Product pins. Article pins. Recipe pins.

## Principle

Use Pinterest Rich Pins to display enhanced metadata on pins. Provide structured data that Pinterest uses for rich previews. Optimize visual content for Pinterest's discovery-focused platform.

## What are Rich Pins?

**Rich Pins:** Enhanced Pinterest pins that show extra metadata from your website

**Types:**
1. **Article Pins** - Blog posts with headline, author, description
2. **Product Pins** - Products with price, availability, purchase link
3. **Recipe Pins** - Recipes with ingredients, cook time, servings

**Benefits:**
- More context for pinners
- Higher engagement and click-through
- Automatic updates when page changes
- Professional appearance
- Better discoverability

## How Pinterest Reads Metadata

**Pinterest uses (in priority order):**
1. Schema.org structured data (JSON-LD)
2. Open Graph tags
3. oEmbed
4. Standard HTML meta tags

**Pinterest does NOT have proprietary meta tags** (unlike older implementations).

## Article Rich Pins

### Required Metadata

**Open Graph:**

```html
<meta property="og:type" content="article">
<meta property="og:title" content="10 Essential Hugo Performance Tips">
<meta property="og:description" content="Learn 10 proven techniques to make your Hugo site faster">
<meta property="og:url" content="https://example.com/blog/hugo-tips/">
<meta property="og:image" content="https://example.com/images/pinterest/hugo-tips.jpg">
<meta property="og:site_name" content="Hugo Best Practices">
<meta property="article:published_time" content="2026-02-13T10:00:00Z">
<meta property="article:author" content="Jane Doe">
```

**Schema.org (preferred by Pinterest):**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "10 Essential Hugo Performance Tips",
  "image": "https://example.com/images/pinterest/hugo-tips.jpg",
  "datePublished": "2026-02-13T10:00:00Z",
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  },
  "publisher": {
    "@type": "Organization",
    "name": "Hugo Best Practices"
  },
  "description": "Learn 10 proven techniques..."
}
</script>
```

## Product Rich Pins

### Required Metadata

**Schema.org Product:**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Hugo Pro Theme",
  "image": "https://example.com/images/products/hugo-pro.jpg",
  "description": "Professional Hugo theme with 50+ components",
  "brand": {
    "@type": "Organization",
    "name": "Hugo Themes"
  },
  "offers": {
    "@type": "Offer",
    "price": "49.00",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock",
    "url": "https://example.com/themes/hugo-pro/"
  }
}
</script>
```

**Displays on Pinterest:**
- Product name
- Current price
- Availability (In Stock / Out of Stock)
- Purchase link

## Recipe Rich Pins

### Required Metadata

**Schema.org Recipe:**

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Recipe",
  "name": "Quick Hugo Site Setup",
  "image": "https://example.com/images/recipes/hugo-setup.jpg",
  "author": {
    "@type": "Person",
    "name": "Jane Doe"
  },
  "datePublished": "2026-02-13",
  "description": "Set up a production Hugo site in 30 minutes",
  "prepTime": "PT10M",
  "cookTime": "PT20M",
  "totalTime": "PT30M",
  "recipeYield": "1 website",
  "recipeIngredient": [
    "Hugo v0.120+",
    "Git",
    "Domain name"
  ],
  "recipeInstructions": [
    {
      "@type": "HowToStep",
      "text": "Install Hugo on your system"
    },
    {
      "@type": "HowToStep",
      "text": "Create a new site with hugo new site"
    }
  ]
}
</script>
```

## Pinterest Image Specifications

### Optimal Pin Image

**Recommended:** 1000×1500 pixels (2:3 ratio)

**Why 2:3?**
- Pinterest is vertically oriented
- Tall images get more visibility in feed
- 2:3 ratio displays fully without cropping

**Requirements:**
- Minimum: 600×900 pixels
- Recommended: 1000×1500 pixels
- Maximum: 6000 pixels (longest dimension)
- File size: Under 20MB
- Formats: JPG, PNG

**Other accepted ratios:**
- 2:3 (1000×1500) - Optimal
- 1:1 (1000×1000) - Square
- 4:5 (1000×1250) - Near optimal
- 1:2.1 (1000×2100) - Maximum height ratio

### Hugo Image Processing

```go-html-template
{{/* Pinterest-optimized image (2:3 ratio) */}}
{{ $image := resources.Get "images/hero.jpg" }}
{{ $pinterest := $image.Fill "1000x1500 center jpg q85" }}

{{/* Include as og:image for Pinterest */}}
<meta property="og:image" content="{{ $pinterest.Permalink }}">
```

### Separate Pinterest Image

**Provide Pinterest-specific image alongside standard OG image:**

```go-html-template
{{/* Standard OG image (1.91:1 for Facebook/Twitter/LinkedIn) */}}
{{ with .Params.images }}
  <meta property="og:image" content="{{ index . 0 | absURL }}">
{{ end }}

{{/* Pinterest image can be different (hidden, used by Save button) */}}
{{ with .Params.pinterest_image }}
  {{/* Pinterest will use this if added to page as hidden image */}}
{{ end }}
```

**Hidden Pinterest image in page body:**

```go-html-template
{{ with .Params.pinterest_image }}
<div style="display:none">
  <img src="{{ . | absURL }}" alt="{{ $.Title }}" data-pin-description="{{ $.Description }}">
</div>
{{ end }}
```

## Pinterest Data Attributes

### Pin It Button Customization

**data-pin attributes on images:**

```html
<img src="/images/photo.jpg"
  data-pin-url="https://example.com/blog/post/"
  data-pin-media="https://example.com/images/pinterest/post.jpg"
  data-pin-description="10 Essential Hugo Performance Tips - Learn to optimize your static site">
```

**Attributes:**
- `data-pin-url` - Override link URL
- `data-pin-media` - Override pin image
- `data-pin-description` - Override pin description
- `data-pin-id` - Existing pin ID (for re-pins)
- `data-pin-nopin="true"` - Prevent pinning this image

### Hugo Template

```go-html-template
{{ with .Params.images }}
  {{ range . }}
    <img src="{{ . | absURL }}"
      alt="{{ $.Title }}"
      data-pin-url="{{ $.Permalink }}"
      data-pin-description="{{ $.Title }} - {{ $.Description }}">
  {{ end }}
{{ end }}
```

## Domain Verification

### Verify Your Website

**Required for Rich Pins:**

1. **Add verification meta tag:**

```html
<meta name="p:domain_verify" content="your-verification-code">
```

2. **Hugo implementation:**

**config.toml:**

```toml
[params]
  pinterest_domain_verify = "your-verification-code"
```

**Template:**

```go-html-template
{{ with .Site.Params.pinterest_domain_verify }}
  <meta name="p:domain_verify" content="{{ . }}">
{{ end }}
```

3. **Verify in Pinterest Business:**
   - Go to Pinterest Business settings
   - Click "Claim website"
   - Enter your domain
   - Choose HTML tag method
   - Verify

### Apply for Rich Pins

**After domain verification:**
1. Go to Pinterest Rich Pin Validator
2. Enter a page URL from your site
3. Validate structured data
4. Apply for Rich Pins
5. Wait for approval (usually instant for articles)

## Hugo Implementation

### Complete Template

**layouts/partials/head/pinterest.html:**

```go-html-template
{{/* Pinterest domain verification */}}
{{ with .Site.Params.pinterest_domain_verify }}
  <meta name="p:domain_verify" content="{{ . }}">
{{ end }}

{{/* Pinterest relies on Open Graph and Schema.org */}}
{{/* Ensure these are included via other partials */}}

{{/* Pinterest-specific description (optional override) */}}
{{ with .Params.pinterest_description }}
  {{/* Store for use in data-pin-description attributes */}}
{{ end }}
```

### Front Matter

```yaml
---
title: "10 Essential Hugo Performance Tips"
description: "Learn 10 proven techniques to make your Hugo site faster"
images:
  - /images/blog/hugo-performance.jpg
pinterest_image: /images/pinterest/hugo-performance-tall.jpg
pinterest_description: "10 Essential Hugo Performance Tips - Learn to make your Hugo site 3x faster with these proven optimization techniques. #Hugo #WebDev #StaticSites"
---
```

## Best Practices

### Images

**✅ DO:**
- Use 2:3 aspect ratio (1000×1500)
- Include text overlay on images
- Use bright, high-contrast colors
- Include brand logo
- Create Pinterest-specific tall images
- Use high-quality photography/graphics

**❌ DON'T:**
- Use landscape images (get cropped)
- Include small, illegible text
- Use dark or low-contrast images
- Forget branding
- Use images under 600px wide

### Descriptions

**✅ DO:**
- Include relevant keywords
- Use hashtags (2-5 per pin)
- Write 100-500 character descriptions
- Include call to action
- Be specific about content value

**❌ DON'T:**
- Stuff keywords
- Use too many hashtags (>20)
- Write vague descriptions
- Skip keywords entirely

### Rich Pins

**✅ DO:**
- Verify domain first
- Use Schema.org structured data
- Include all required fields
- Test with Rich Pin Validator
- Keep structured data accurate

**❌ DON'T:**
- Skip domain verification
- Use outdated metadata
- Include inaccurate pricing
- Forget to update when content changes

## Guidelines

### Essential

**Minimum for Pinterest:**
- Domain verification (p:domain_verify)
- og:title, og:description, og:image
- Schema.org Article or Product data
- Images at least 600×900

### Recommended

**For better Pinterest performance:**
- 1000×1500 Pinterest-specific images
- data-pin-description on images
- Rich Pin validation
- Hashtags in descriptions
- Text overlay on images

### Advanced

**For maximum Pinterest traffic:**
- Separate Pinterest images per post
- Hidden Pinterest-optimized images
- Product Rich Pins with pricing
- Recipe Rich Pins
- Pinterest Analytics integration
- A/B testing pin designs

## Benefits

Visual Discovery. Enhanced pins stand out in Pinterest feed.

Automatic Updates. Rich Pins sync with your website data.

Credibility. Verified domain and rich metadata build trust.

Traffic. Pinterest drives significant referral traffic for visual content.

SEO. Pinterest pins rank in Google Image Search.

## Related

- [open-graph-meta-tags.md](./open-graph-meta-tags.md) - Open Graph tags (used by Pinterest)
- [structured-data-schema-org.md](./structured-data-schema-org.md) - Schema.org (preferred by Pinterest)
- [social-media-image-generation.md](./social-media-image-generation.md) - Image generation
- [twitter-card-images.md](./twitter-card-images.md) - Social media image specs
