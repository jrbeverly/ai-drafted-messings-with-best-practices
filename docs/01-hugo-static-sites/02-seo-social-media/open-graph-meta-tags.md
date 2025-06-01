# Open Graph Meta Tags

Open Graph Protocol. Social media previews. Facebook sharing. Link previews. OG tags best practices.

## Principle

Use Open Graph meta tags to control how content appears when shared on social media. Provide rich previews with images, titles, and descriptions. Improve click-through rates from social platforms.

## What is Open Graph?

**Open Graph Protocol:** Metadata standard created by Facebook

**Purpose:** Control social media link previews

**Used by:**
- Facebook
- LinkedIn
- Twitter (as fallback)
- Pinterest
- Discord
- Slack
- WhatsApp
- iMessage

**Specification:** https://ogp.me/

## Required Tags

### og:title

```html
<meta property="og:title" content="Page Title">
```

**Best practices:**
- 60-90 characters optimal
- Front-load important keywords
- Don't include site name (Facebook adds it)
- Make it compelling and clickable

**Example:**
```html
<meta property="og:title" content="10 Essential Hugo Performance Tips">
```

### og:type

```html
<meta property="og:type" content="website">
```

**Common types:**
- `website` - General pages, homepage
- `article` - Blog posts, news articles
- `profile` - Personal profiles
- `video.movie` - Video content
- `music.song` - Audio content
- `book` - Books, publications

**Example:**
```html
<!-- Homepage -->
<meta property="og:type" content="website">

<!-- Blog post -->
<meta property="og:type" content="article">
```

### og:url

```html
<meta property="og:url" content="https://example.com/page">
```

**Requirements:**
- Absolute URL (not relative)
- Canonical URL (not paginated or variant)
- HTTPS preferred
- Include trailing slash consistently

**Example:**
```html
<meta property="og:url" content="https://example.com/blog/post-title/">
```

### og:image

```html
<meta property="og:image" content="https://example.com/images/og-image.jpg">
```

**Requirements:**
- Absolute URL
- Accessible without authentication
- Recommended size: 1200×630 pixels
- Minimum size: 200×200 pixels
- Maximum size: 8MB
- Supported formats: JPG, PNG, WebP

**Example:**
```html
<meta property="og:image" content="https://example.com/images/blog/post-og-image.jpg">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:alt" content="Descriptive text for image">
```

## Recommended Tags

### og:description

```html
<meta property="og:description" content="Compelling description that makes people want to click">
```

**Best practices:**
- 155-200 characters optimal
- First 100 characters most important
- Include value proposition
- End with call-to-action

**Example:**
```html
<meta property="og:description" content="Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples.">
```

### og:site_name

```html
<meta property="og:site_name" content="Site Name">
```

**Purpose:** Displayed alongside page title

**Example:**
```html
<meta property="og:site_name" content="Hugo Best Practices">
```

### og:locale

```html
<meta property="og:locale" content="en_US">
```

**Format:** language_TERRITORY

**Examples:**
```html
<meta property="og:locale" content="en_US">
<meta property="og:locale" content="es_ES">
<meta property="og:locale" content="fr_FR">
<meta property="og:locale" content="de_DE">
```

## Complete Example

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <title>10 Essential Hugo Performance Tips | Hugo Best Practices</title>
  <meta name="description" content="Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples.">

  <!-- Open Graph / Facebook -->
  <meta property="og:type" content="article">
  <meta property="og:url" content="https://example.com/blog/hugo-performance-tips/">
  <meta property="og:title" content="10 Essential Hugo Performance Tips">
  <meta property="og:description" content="Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples.">
  <meta property="og:image" content="https://example.com/images/blog/hugo-performance-og.jpg">
  <meta property="og:image:width" content="1200">
  <meta property="og:image:height" content="630">
  <meta property="og:image:alt" content="Hugo performance optimization dashboard showing speed improvements">
  <meta property="og:site_name" content="Hugo Best Practices">
  <meta property="og:locale" content="en_US">
</head>
<body>
  <!-- Content -->
</body>
</html>
```

## Hugo Implementation

### Hugo Template

**layouts/partials/head/opengraph.html:**

```go-html-template
{{/* Open Graph */}}
<meta property="og:type" content="{{ if .IsPage }}article{{ else }}website{{ end }}">
<meta property="og:url" content="{{ .Permalink }}">
<meta property="og:title" content="{{ .Title }}">
<meta property="og:description" content="{{ with .Description }}{{ . }}{{ else }}{{ .Site.Params.description }}{{ end }}">

{{ with .Params.images }}
  {{ range first 1 . }}
    <meta property="og:image" content="{{ . | absURL }}">
  {{ end }}
{{ else }}
  {{ with $.Site.Params.og_image }}
    <meta property="og:image" content="{{ . | absURL }}">
  {{ end }}
{{ end }}

{{ with .Params.og_image_width }}
  <meta property="og:image:width" content="{{ . }}">
{{ end }}

{{ with .Params.og_image_height }}
  <meta property="og:image:height" content="{{ . }}">
{{ end }}

{{ with .Params.og_image_alt }}
  <meta property="og:image:alt" content="{{ . }}">
{{ end }}

<meta property="og:site_name" content="{{ .Site.Title }}">
<meta property="og:locale" content="{{ .Site.Language.Lang | replace "-" "_" }}">
```

### Content Front Matter

```yaml
---
title: "10 Essential Hugo Performance Tips"
description: "Learn 10 proven techniques to make your Hugo site load 3x faster."
images:
  - /images/blog/hugo-performance-og.jpg
og_image_width: 1200
og_image_height: 630
og_image_alt: "Hugo performance optimization dashboard"
---
```

## Image Best Practices

### Optimal Dimensions

**Recommended size:** 1200×630 pixels (1.91:1 ratio)

**Why:**
- Facebook: Displays perfectly
- LinkedIn: Full size support
- Twitter: Falls back correctly

**Minimum acceptable:** 600×315 pixels

**Other common sizes:**
- 1200×1200 (square, Instagram)
- 1200×900 (4:3)
- 1080×1920 (9:16, stories)

### Design Guidelines

**✅ DO:**
- Use high-quality images
- Include readable text (minimum 32px font)
- High contrast colors
- Brand logo in corner
- Focus subject in center
- Save as JPG (smaller file size)

**❌ DON'T:**
- Small, illegible text
- Complex graphics
- Too much text (< 20% of image)
- Low resolution
- Clickbait imagery

### File Size

**Target:** < 300KB

**Optimization:**
```bash
# Resize and optimize with ImageMagick
convert input.jpg -resize 1200x630^ -gravity center -extent 1200x630 -quality 85 og-image.jpg

# Or with Hugo image processing
{{ $image := resources.Get "images/hero.jpg" }}
{{ $og := $image.Fill "1200x630 center jpg q85" }}
<meta property="og:image" content="{{ $og.Permalink }}">
```

## Testing Open Graph Tags

### Facebook Sharing Debugger

**URL:** https://developers.facebook.com/tools/debug/

**How to use:**
1. Enter page URL
2. Click "Debug"
3. View how Facebook sees your page
4. Check for errors/warnings
5. Click "Scrape Again" after changes

### LinkedIn Post Inspector

**URL:** https://www.linkedin.com/post-inspector/

**How to use:**
1. Enter URL
2. View preview
3. Clear cache if needed

### Twitter Card Validator

**URL:** https://cards-dev.twitter.com/validator

**Note:** Requires Twitter developer account

## Common Issues

### Image Not Showing

**Problem:** OG image not displaying

**Solutions:**
- Use absolute URL (https://example.com/image.jpg)
- Ensure image is publicly accessible
- Check image size (minimum 200×200)
- Verify file format (JPG, PNG, WebP)
- Clear Facebook cache (Sharing Debugger)

### Wrong Image Showing

**Problem:** Old image displayed after update

**Solution:**
```html
<!-- Add cache-busting parameter -->
<meta property="og:image" content="https://example.com/image.jpg?v=2">

<!-- Or use Facebook Sharing Debugger to clear cache -->
```

### Title Too Long

**Problem:** Title truncated

**Solution:**
```html
<!-- Keep under 60 characters -->
<meta property="og:title" content="Hugo Performance: 10 Essential Tips">
```

## Guidelines

**Essential:**
- og:title
- og:type
- og:url
- og:image

**Recommended:**
- og:description
- og:site_name
- og:locale
- og:image:width and height
- og:image:alt

**Image specs:**
- 1200×630 pixels
- < 300KB file size
- JPG or PNG format
- Absolute HTTPS URL

## Benefits

Better Engagement. Rich previews increase clicks.

Brand Control. Consistent social presence.

Professional. Polished link previews.

Universal. Works across platforms.

## Related

- [opengraph-article-extensions.md](./opengraph-article-extensions.md) - Article-specific tags
- [twitter-card-images.md](./twitter-card-images.md) - Twitter-specific images
- [social-media-image-generation.md](./social-media-image-generation.md) - Automated image generation
- [../../01-hugo-basics/hugo-content-management.md](../../01-hugo-static-sites/01-hugo-basics/hugo-content-management.md) - Content front matter
