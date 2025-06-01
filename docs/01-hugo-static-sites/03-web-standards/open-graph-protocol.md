# Open Graph Protocol

Social media link previews. Facebook, LinkedIn, Discord sharing. Rich cards. Structured metadata for social platforms.

## Principle

Control how your content appears when shared on social media. Provide rich previews with images, titles, and descriptions. Improve click-through rates from social platforms.

## What is Open Graph?

**Purpose:** Metadata protocol created by Facebook (now Meta) to control how URLs are displayed when shared.

**Format:** `<meta>` tags in HTML `<head>` with `property="og:*"` attributes

**Supported By:**
- Facebook
- LinkedIn
- Discord
- Slack
- iMessage
- WhatsApp
- Pinterest (partially, prefers Pinterest-specific tags)
- Many other platforms

**Specification:** https://ogp.me/

## Basic Open Graph Tags

```html
<!-- Required Open Graph tags -->
<meta property="og:title" content="Your Page Title">
<meta property="og:type" content="website">
<meta property="og:url" content="https://example.com/page">
<meta property="og:image" content="https://example.com/og-image.jpg">
```

**These four tags are the minimum required.**

## Complete Open Graph Tags

```html
<!-- Essential Open Graph tags -->
<meta property="og:title" content="Complete Guide to Open Graph Protocol">
<meta property="og:type" content="article">
<meta property="og:url" content="https://example.com/guides/open-graph">
<meta property="og:image" content="https://example.com/images/og-open-graph.jpg">
<meta property="og:description" content="Learn how to implement Open Graph Protocol for rich social media link previews on Facebook, LinkedIn, and more.">

<!-- Additional recommended tags -->
<meta property="og:site_name" content="Example Blog">
<meta property="og:locale" content="en_US">

<!-- Image details -->
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:alt" content="Open Graph Protocol diagram showing meta tags">
<meta property="og:image:type" content="image/jpeg">

<!-- Article-specific (for blog posts) -->
<meta property="article:published_time" content="2026-02-13T10:00:00Z">
<meta property="article:modified_time" content="2026-02-13T15:30:00Z">
<meta property="article:author" content="https://example.com/authors/jane-doe">
<meta property="article:section" content="Technology">
<meta property="article:tag" content="Web Development">
<meta property="article:tag" content="Open Graph">
<meta property="article:tag" content="Social Media">
```

## Open Graph Types

**Common types:**

| Type | Usage | Additional Properties |
|------|-------|----------------------|
| `website` | Homepage, static pages | None required |
| `article` | Blog posts, news articles | `article:published_time`, `article:author`, `article:section`, `article:tag` |
| `profile` | User profiles | `profile:first_name`, `profile:last_name`, `profile:username` |
| `video.movie` | Movies | `video:actor`, `video:director`, `video:duration` |
| `video.episode` | TV episodes | `video:series`, `video:duration` |
| `music.song` | Songs | `music:duration`, `music:album`, `music:musician` |
| `book` | Books | `book:author`, `book:isbn`, `book:release_date` |

**Most common:** `website` (default) and `article` (blog posts).

## Image Specifications

**Recommended Image Size:**
- **1200×630 pixels** (1.91:1 aspect ratio) - Facebook, LinkedIn standard
- Minimum: 600×315 pixels
- Maximum: 8 MB file size

**Alternative Sizes:**
- 1200×1200 (square) - Works everywhere
- 1080×1080 (square) - Instagram-style

**Format:**
- JPG or PNG
- Avoid transparency (use solid background)
- High quality, clear visuals

**Design Guidelines:**
- Important content in center (safe area: 1000×525)
- Text should be large (minimum 60px font size)
- High contrast
- Brand logo prominent
- No fine details (will be compressed)

```html
<!-- Recommended image tags -->
<meta property="og:image" content="https://example.com/og-image.jpg">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:alt" content="Descriptive text for image">
<meta property="og:image:type" content="image/jpeg">

<!-- Optional: Multiple images -->
<meta property="og:image" content="https://example.com/og-image-1.jpg">
<meta property="og:image" content="https://example.com/og-image-2.jpg">
```

## Hugo Implementation

### Hugo Template (Head Partial)

```go-html-template
{{/* layouts/partials/head.html */}}

<!-- Open Graph tags -->
<meta property="og:title" content="{{ .Title }}">
<meta property="og:type" content="{{ if .IsPage }}article{{ else }}website{{ end }}">
<meta property="og:url" content="{{ .Permalink }}">

{{- with .Params.image }}
<meta property="og:image" content="{{ . | absURL }}">
{{- else }}
<meta property="og:image" content="{{ "/images/og-default.jpg" | absURL }}">
{{- end }}

{{- with .Description }}
<meta property="og:description" content="{{ . }}">
{{- else }}
<meta property="og:description" content="{{ .Site.Params.description }}">
{{- end }}

<meta property="og:site_name" content="{{ .Site.Title }}">
<meta property="og:locale" content="{{ .Site.Language.Lang | default "en_US" }}">

{{- if .IsPage }}
<!-- Article-specific tags -->
<meta property="article:published_time" content="{{ .PublishDate.Format "2006-01-02T15:04:05Z07:00" }}">
<meta property="article:modified_time" content="{{ .Lastmod.Format "2006-01-02T15:04:05Z07:00" }}">

{{- with .Params.author }}
<meta property="article:author" content="{{ . }}">
{{- end }}

{{- range .Params.tags }}
<meta property="article:tag" content="{{ . }}">
{{- end }}

{{- with .Params.category }}
<meta property="article:section" content="{{ . }}">
{{- end }}
{{- end }}
```

### Content Front Matter

```yaml
---
title: "Complete Guide to Open Graph Protocol"
date: 2026-02-13
description: "Learn how to implement Open Graph Protocol for rich social media link previews."
image: "/images/posts/open-graph-guide.jpg"
author: "Jane Doe"
category: "Web Development"
tags:
  - Open Graph
  - Social Media
  - SEO
---
```

### Advanced Hugo Template (Image Dimensions)

```go-html-template
{{/* layouts/partials/open-graph.html */}}

{{- $image := "" }}
{{- $imageWidth := 1200 }}
{{- $imageHeight := 630 }}

{{- if .Params.image }}
  {{- $image = .Params.image | absURL }}

  {{/* Get image resource to extract dimensions */}}
  {{- with .Resources.GetMatch (.Params.image) }}
    {{- $imageWidth = .Width }}
    {{- $imageHeight = .Height }}
  {{- end }}
{{- else }}
  {{- $image = "/images/og-default.jpg" | absURL }}
{{- end }}

<meta property="og:image" content="{{ $image }}">
<meta property="og:image:width" content="{{ $imageWidth }}">
<meta property="og:image:height" content="{{ $imageHeight }}">
<meta property="og:image:type" content="image/jpeg">

{{- with .Params.imageAlt }}
<meta property="og:image:alt" content="{{ . }}">
{{- else }}
<meta property="og:image:alt" content="{{ .Title }}">
{{- end }}
```

## Dynamic OG Images (Hugo Shortcode)

```go-html-template
{{/* layouts/shortcodes/og-image.html */}}
{{/* Generate dynamic OG image URL */}}

{{- $title := .Get "title" | default $.Page.Title }}
{{- $description := .Get "description" | default $.Page.Description }}

{{/* Use external service like og-image.vercel.app or self-hosted */}}
{{- $ogImageURL := printf "https://og-image.example.com/?title=%s&description=%s"
    (urlquery $title)
    (urlquery $description) }}

<meta property="og:image" content="{{ $ogImageURL }}">
```

**Usage in content:**

```markdown
{{< og-image title="My Post" description="Post description" >}}
```

## Article-Specific Tags

```html
<!-- For blog posts and articles -->
<meta property="og:type" content="article">

<!-- Publishing info -->
<meta property="article:published_time" content="2026-02-13T10:00:00Z">
<meta property="article:modified_time" content="2026-02-13T15:30:00Z">
<meta property="article:expiration_time" content="2027-02-13T10:00:00Z">

<!-- Author -->
<meta property="article:author" content="Jane Doe">
<meta property="article:author" content="https://example.com/authors/jane-doe">

<!-- Categorization -->
<meta property="article:section" content="Technology">

<!-- Tags (multiple allowed) -->
<meta property="article:tag" content="Web Development">
<meta property="article:tag" content="Open Graph">
<meta property="article:tag" content="SEO">
```

**Hugo Template:**

```go-html-template
{{- if eq .Type "post" }}
<meta property="og:type" content="article">
<meta property="article:published_time" content="{{ .PublishDate.Format "2006-01-02T15:04:05Z07:00" }}">
<meta property="article:modified_time" content="{{ .Lastmod.Format "2006-01-02T15:04:05Z07:00" }}">

{{- with .Params.author }}
<meta property="article:author" content="{{ . }}">
{{- end }}

{{- range .Params.tags }}
<meta property="article:tag" content="{{ . }}">
{{- end }}
{{- end }}
```

## Video and Audio

```html
<!-- Video -->
<meta property="og:type" content="video.other">
<meta property="og:video" content="https://example.com/video.mp4">
<meta property="og:video:secure_url" content="https://example.com/video.mp4">
<meta property="og:video:type" content="video/mp4">
<meta property="og:video:width" content="1920">
<meta property="og:video:height" content="1080">

<!-- Audio -->
<meta property="og:type" content="music.song">
<meta property="og:audio" content="https://example.com/song.mp3">
<meta property="og:audio:secure_url" content="https://example.com/song.mp3">
<meta property="og:audio:type" content="audio/mpeg">
```

## Testing Open Graph Tags

### Facebook Sharing Debugger

**URL:** https://developers.facebook.com/tools/debug/

**How to use:**
1. Enter your URL
2. Click "Debug"
3. Review preview
4. Click "Scrape Again" to refresh cache

**Fix issues:**
- Missing required tags
- Invalid image URLs
- Incorrect image dimensions
- Wrong MIME types

### LinkedIn Post Inspector

**URL:** https://www.linkedin.com/post-inspector/

**How to use:**
1. Enter your URL
2. View preview
3. "Inspect" button shows raw data

### Twitter Card Validator (also checks OG)

**URL:** https://cards-dev.twitter.com/validator

**Note:** Twitter uses its own Twitter Cards, but falls back to Open Graph tags.

### Discord Embed Visualizer (Unofficial)

**URL:** https://leovoel.github.io/embed-visualizer/

### Manual Testing

```bash
# Check if OG tags exist
curl https://example.com | grep "og:"

# Extract OG tags
curl https://example.com | grep -o '<meta property="og:[^"]*" content="[^"]*"'

# Validate image URL
curl -I https://example.com/og-image.jpg
# Should return 200 OK
```

## Common Platforms and Variations

### Facebook

**Minimum required:**
- `og:title`
- `og:type`
- `og:url`
- `og:image`

**Recommended:**
- `og:description`
- `og:site_name`
- `og:image:width` and `og:image:height`

### LinkedIn

**Uses Open Graph tags** (same as Facebook)

**Best practices:**
- Professional tone in description
- High-quality images (company branding)
- Article type for blog posts

### Discord

**Uses Open Graph + custom `theme-color`**

```html
<!-- Discord-specific -->
<meta property="og:title" content="Discord Post Title">
<meta property="og:description" content="Description shown in embed">
<meta property="og:image" content="https://example.com/og-image.jpg">
<meta name="theme-color" content="#2196F3">
```

**Theme color:** Accent color for Discord embed.

### Slack

**Uses Open Graph tags**

**Fallback:** If no OG tags, uses `<title>` and `<meta name="description">`.

### iMessage / WhatsApp

**Uses Open Graph tags** for rich link previews.

## Fallback Tags (Non-OG)

```html
<!-- Standard HTML tags (fallback if OG missing) -->
<title>Page Title</title>
<meta name="description" content="Page description">
<link rel="image_src" href="/image.jpg">  <!-- Old fallback -->

<!-- Open Graph (preferred) -->
<meta property="og:title" content="Page Title">
<meta property="og:description" content="Page description">
<meta property="og:image" content="/og-image.jpg">
```

**Recommendation:** Always include both standard HTML tags AND Open Graph tags.

## Dynamic OG Images (Server-Side Generation)

### Option 1: Vercel OG Image

```html
<meta property="og:image" content="https://og-image.vercel.app/My%20Title.png?theme=light&md=1&fontSize=100px">
```

**Service:** https://github.com/vercel/og-image

### Option 2: Cloudinary

```html
<meta property="og:image" content="https://res.cloudinary.com/demo/image/upload/l_text:Arial_100:My%20Title/og-template.jpg">
```

### Option 3: Self-Hosted (AWS Lambda + Puppeteer)

```javascript
// Lambda function to generate OG images
const puppeteer = require('puppeteer-core');
const chromium = require('@sparticuz/chromium');

exports.handler = async (event) => {
  const { title, description } = event.queryStringParameters;

  const browser = await puppeteer.launch({
    args: chromium.args,
    executablePath: await chromium.executablePath(),
  });

  const page = await browser.newPage();
  await page.setViewport({ width: 1200, height: 630 });

  await page.setContent(`
    <html>
      <style>
        body {
          margin: 0;
          padding: 80px;
          font-family: Arial;
          background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
          color: white;
        }
        h1 { font-size: 80px; margin: 0; }
        p { font-size: 40px; margin-top: 20px; opacity: 0.9; }
      </style>
      <body>
        <h1>${title}</h1>
        <p>${description}</p>
      </body>
    </html>
  `);

  const screenshot = await page.screenshot({ type: 'jpeg', quality: 90 });
  await browser.close();

  return {
    statusCode: 200,
    headers: { 'Content-Type': 'image/jpeg' },
    body: screenshot.toString('base64'),
    isBase64Encoded: true,
  };
};
```

## Common Mistakes

❌ **Relative URLs for images:**
```html
<!-- Wrong -->
<meta property="og:image" content="/images/og-image.jpg">

<!-- Correct (absolute URL required) -->
<meta property="og:image" content="https://example.com/images/og-image.jpg">
```

❌ **Missing required tags:**
```html
<!-- Wrong (missing og:url and og:type) -->
<meta property="og:title" content="Title">
<meta property="og:image" content="https://example.com/image.jpg">

<!-- Correct (all four required) -->
<meta property="og:title" content="Title">
<meta property="og:type" content="website">
<meta property="og:url" content="https://example.com">
<meta property="og:image" content="https://example.com/image.jpg">
```

❌ **Image too small:**
```html
<!-- Wrong (image too small, won't display well) -->
<meta property="og:image" content="https://example.com/tiny-logo.png">
<!-- Image is 200x200, minimum is 600x315 -->

<!-- Correct -->
<meta property="og:image" content="https://example.com/og-image.jpg">
<!-- Image is 1200x630 -->
```

❌ **Using `name` instead of `property`:**
```html
<!-- Wrong -->
<meta name="og:title" content="Title">

<!-- Correct -->
<meta property="og:title" content="Title">
```

❌ **Not testing on actual platforms:**
```
# Don't just assume it works - test it!
# Use Facebook Sharing Debugger, LinkedIn Post Inspector, etc.
```

## Best Practices

**Required:**
- All four essential tags (title, type, url, image)
- Absolute URLs (especially for images)
- Image size 1200×630 or larger
- High-quality, relevant images

**Recommended:**
- `og:description` (80-200 characters)
- `og:site_name`
- `og:locale`
- Image dimensions (`og:image:width`, `og:image:height`)
- Image alt text (`og:image:alt`)

**Testing:**
- Test on actual platforms (Facebook, LinkedIn)
- Use debugging tools
- Check mobile and desktop
- Verify images load correctly

**Maintenance:**
- Update OG tags when content changes
- Clear cache on social platforms after updates
- Monitor broken images

## Guidelines

**Essential Tags (Required):**
```html
<meta property="og:title" content="...">
<meta property="og:type" content="...">
<meta property="og:url" content="...">
<meta property="og:image" content="...">
```

**Recommended Tags:**
```html
<meta property="og:description" content="...">
<meta property="og:site_name" content="...">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
```

**Image Requirements:**
- 1200×630 pixels (recommended)
- Absolute URL (https://)
- JPG or PNG format
- < 8 MB file size

## Benefits

Shareable. Rich link previews on social media.

Professional. Polished appearance when shared.

Clickable. Higher engagement from previews.

Controlled. You decide how links appear.

## Related

- [twitter-cards.md](./twitter-cards.md) - Twitter-specific cards
- [meta-tags-seo.md](./meta-tags-seo.md) - HTML meta tags
- [structured-data-json-ld.md](./structured-data-json-ld.md) - Structured data
- [schema-org-markup.md](./schema-org-markup.md) - Schema.org
