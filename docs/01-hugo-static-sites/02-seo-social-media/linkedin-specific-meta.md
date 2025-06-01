# LinkedIn-Specific Meta Tags

LinkedIn sharing optimization. Professional content preview. Company page integration. Article publishing meta.

## Principle

Optimize content previews for LinkedIn sharing. Use Open Graph tags that LinkedIn prioritizes. Design professional-looking share previews for B2B and career-focused audiences.

## How LinkedIn Uses Meta Tags

**LinkedIn primarily uses Open Graph tags** for link previews.

**LinkedIn reads:**
1. Open Graph tags (primary)
2. Twitter Card tags (fallback)
3. Standard meta tags (last resort)

**No proprietary LinkedIn meta tags exist** - use Open Graph for optimal results.

## Required Tags for LinkedIn

### Open Graph (Used by LinkedIn)

```html
<meta property="og:title" content="10 Essential Hugo Performance Tips">
<meta property="og:description" content="Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide for developers.">
<meta property="og:image" content="https://example.com/images/linkedin-share.jpg">
<meta property="og:url" content="https://example.com/blog/hugo-performance-tips/">
<meta property="og:type" content="article">
```

### Image Specifications for LinkedIn

**Recommended:** 1200×627 pixels (1.91:1 ratio)

**Requirements:**
- Minimum: 200×200 pixels (appears as small square)
- Recommended: 1200×627 pixels (large landscape preview)
- Maximum file size: 5MB
- Formats: JPG, PNG, GIF
- Aspect ratio: 1.91:1 for large preview

**Display sizes:**
- Feed (large): 552×289 pixels
- Feed (small): 80×80 pixels (if image < 400px wide)

**Key point:** Images under 400px wide display as small square thumbnails. Always use 1200×627.

### Article Tags

```html
<meta property="og:type" content="article">
<meta property="article:published_time" content="2026-02-13T10:00:00Z">
<meta property="article:modified_time" content="2026-02-14T09:15:00Z">
<meta property="article:author" content="https://www.linkedin.com/in/janedoe/">
<meta property="article:section" content="Web Development">
<meta property="article:tag" content="Hugo">
<meta property="article:tag" content="Performance">
```

## Hugo Implementation

### LinkedIn-Optimized Template

**layouts/partials/head/linkedin-meta.html:**

```go-html-template
{{/* Open Graph tags optimized for LinkedIn */}}
<meta property="og:type" content="{{ if .IsPage }}article{{ else }}website{{ end }}">
<meta property="og:url" content="{{ .Permalink }}">
<meta property="og:title" content="{{ .Title }}">
<meta property="og:description" content="{{ with .Description }}{{ . }}{{ else }}{{ .Summary | plainify | truncate 200 }}{{ end }}">
<meta property="og:site_name" content="{{ .Site.Title }}">
<meta property="og:locale" content="{{ replace .Site.Language.Lang "-" "_" }}">

{{/* Image - LinkedIn needs 1200x627 minimum for large preview */}}
{{ $ogImage := "" }}
{{ with .Params.linkedin_image }}
  {{ $ogImage = . | absURL }}
{{ else }}
  {{ with .Params.images }}
    {{ $img := index . 0 }}
    {{ $resource := resources.Get $img }}
    {{ if $resource }}
      {{ $linkedin := $resource.Fill "1200x627 center jpg q85" }}
      {{ $ogImage = $linkedin.Permalink }}
    {{ else }}
      {{ $ogImage = $img | absURL }}
    {{ end }}
  {{ else }}
    {{ with .Site.Params.og_image }}
      {{ $ogImage = . | absURL }}
    {{ end }}
  {{ end }}
{{ end }}

{{ with $ogImage }}
  <meta property="og:image" content="{{ . }}">
  <meta property="og:image:width" content="1200">
  <meta property="og:image:height" content="627">
{{ end }}

{{/* Article metadata */}}
{{ if .IsPage }}
  <meta property="article:published_time" content="{{ .Date.Format "2006-01-02T15:04:05Z07:00" }}">
  {{ if ne .Lastmod .Date }}
    <meta property="article:modified_time" content="{{ .Lastmod.Format "2006-01-02T15:04:05Z07:00" }}">
  {{ end }}
  {{ with .Params.linkedin_author }}
    <meta property="article:author" content="https://www.linkedin.com/in/{{ . }}/">
  {{ end }}
  {{ with .Section }}
    <meta property="article:section" content="{{ . | humanize }}">
  {{ end }}
  {{ range .Params.tags }}
    <meta property="article:tag" content="{{ . }}">
  {{ end }}
{{ end }}
```

### Content Front Matter

```yaml
---
title: "10 Essential Hugo Performance Tips"
description: "Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide for developers."
date: 2026-02-13T10:00:00Z
images:
  - /images/blog/hugo-performance.jpg
linkedin_author: janedoe
linkedin_image: /images/linkedin/hugo-performance.jpg
tags:
  - Hugo
  - Performance
  - Web Development
---
```

## LinkedIn Post Inspector

### Cache Clearing

**LinkedIn caches shared link previews aggressively.**

**LinkedIn Post Inspector:** https://www.linkedin.com/post-inspector/

**How to use:**
1. Enter page URL
2. Click "Inspect"
3. View how LinkedIn renders preview
4. Click "Refresh" to clear cache

**When to use:**
- After updating OG tags
- After changing share image
- Before sharing important content
- To verify preview looks correct

### Common Preview Issues

**Small thumbnail instead of large image:**
- Image must be 1200×627 minimum
- Image must be publicly accessible
- URL must use HTTPS

**Wrong title or description:**
- Check og:title and og:description tags
- Clear cache with Post Inspector
- Wait 7 days for full cache refresh

## Best Practices

### Content Optimization

**✅ DO:**
- Write professional, B2B-focused descriptions
- Include industry keywords
- Keep title under 70 characters
- Keep description under 200 characters
- Use 1200×627 pixel images
- Include professional headshots for author content
- Link to LinkedIn profiles in article:author

**❌ DON'T:**
- Use casual/clickbait language
- Include emojis in OG tags (can in post text)
- Use images smaller than 400px wide
- Forget to clear LinkedIn cache after updates
- Use stock photos that look generic

### Image Design for LinkedIn

**✅ DO:**
- Use clean, professional design
- Include readable text (white on dark)
- Add company branding
- Use high-contrast colors
- Test at 552×289 (feed display size)
- Keep important content centered

**❌ DON'T:**
- Use busy backgrounds
- Include small text
- Use low-resolution images
- Place text near edges
- Use unprofessional imagery

### LinkedIn-Specific Tips

**Title optimization:**
- Lead with value proposition
- Include numbers ("10 Tips", "5 Ways")
- Use industry terms
- Professional tone

**Description optimization:**
- First 100 characters most visible
- Include call-to-action
- Mention specific outcomes
- Professional language

## Testing

### LinkedIn Post Inspector

```
URL: https://www.linkedin.com/post-inspector/
1. Enter page URL
2. Click "Inspect"
3. Verify title, description, image
4. Click "Refresh" to clear cache
```

### Manual Check

```bash
# Check OG tags
curl -s https://example.com/blog/post/ | grep 'property="og:'

# Verify image is accessible
curl -I https://example.com/images/linkedin-share.jpg

# Check image dimensions
identify https://example.com/images/linkedin-share.jpg
```

## Guidelines

### Essential

**Minimum for LinkedIn sharing:**
- og:title (under 70 characters)
- og:description (under 200 characters)
- og:image (1200×627 pixels, HTTPS)
- og:url (canonical URL)
- og:type (article for posts)

### Recommended

**For better engagement:**
- article:published_time
- article:author (LinkedIn profile URL)
- article:tag
- Professional image design
- Cache clearing via Post Inspector

### Advanced

**For maximum LinkedIn presence:**
- LinkedIn-specific image (separate from OG)
- Company page integration
- Author profile linking
- Industry-specific keywords
- A/B testing share images

## Benefits

Professional Sharing. Polished previews for B2B audience.

Brand Visibility. Consistent branding on professional network.

Engagement. Well-designed previews increase click-through.

Authority. Professional appearance builds credibility.

Reach. LinkedIn has 900M+ professionals globally.

## Related

- [open-graph-meta-tags.md](./open-graph-meta-tags.md) - Open Graph Protocol (LinkedIn's primary source)
- [opengraph-article-extensions.md](./opengraph-article-extensions.md) - Article-specific metadata
- [social-media-image-generation.md](./social-media-image-generation.md) - Automated image generation
- [twitter-card-images.md](./twitter-card-images.md) - Twitter Card images (similar specs)
