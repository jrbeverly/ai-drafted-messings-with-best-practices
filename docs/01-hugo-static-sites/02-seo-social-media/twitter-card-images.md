# Twitter Card Images

Twitter Card images. Social media previews. Image specifications. Optimal dimensions. Design best practices.

## Principle

Use Twitter Card images to control how content appears when shared on Twitter/X. Provide visually appealing previews that increase engagement. Optimize images for Twitter's specific display requirements.

## What are Twitter Cards?

**Twitter Cards:** Rich media attachments for tweets containing links

**Purpose:** Enhanced link previews with images, titles, and descriptions

**Benefits:**
- Higher engagement (2-3x more clicks)
- Professional brand presence
- Consistent visual identity
- Better mobile experience

**Card Types:**
- `summary` - Small image (1:1 ratio)
- `summary_large_image` - Large image (2:1 ratio) **← Most common**
- `app` - Mobile app install cards
- `player` - Video/audio player cards

## Summary Large Image Card

**Most popular card type for content sharing.**

### Basic Implementation

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@yourusername">
<meta name="twitter:creator" content="@authorusername">
<meta name="twitter:title" content="Page Title">
<meta name="twitter:description" content="Page description">
<meta name="twitter:image" content="https://example.com/images/twitter-card.jpg">
```

### Required Tags

**twitter:card**
```html
<meta name="twitter:card" content="summary_large_image">
```

**twitter:title**
```html
<meta name="twitter:title" content="10 Essential Hugo Performance Tips">
```
- Maximum 70 characters
- Front-load keywords
- Make it compelling

**twitter:description**
```html
<meta name="twitter:description" content="Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples.">
```
- Maximum 200 characters
- First 100 characters most important
- Include call-to-action

**twitter:image**
```html
<meta name="twitter:image" content="https://example.com/images/blog/hugo-performance-twitter.jpg">
```
- Absolute URL (HTTPS)
- Publicly accessible
- See image specifications below

### Optional Tags

**twitter:site**
```html
<meta name="twitter:site" content="@yourbrand">
```
- Website's Twitter handle
- Displays "via @yourbrand"

**twitter:creator**
```html
<meta name="twitter:creator" content="@authorhandle">
```
- Content author's Twitter handle
- Displays author attribution

**twitter:image:alt**
```html
<meta name="twitter:image:alt" content="Dashboard showing Hugo site performance improvements">
```
- Accessibility description
- Maximum 420 characters
- Required for accessibility

## Image Specifications

### Summary Large Image

**Optimal dimensions:** 1200×628 pixels (1.91:1 ratio)

**Requirements:**
- Minimum size: 300×157 pixels
- Maximum size: 4096×4096 pixels
- Maximum file size: 5MB
- Supported formats: JPG, PNG, WebP, GIF
- Aspect ratio: 1.91:1 (same as Facebook)

**Why 1200×628?**
- Displays perfectly on desktop and mobile
- Compatible with Open Graph images
- Maintains quality when scaled

**Display size:**
- Desktop: 506×266 pixels
- Mobile: Full width, proportional height

### Summary Card (Small Image)

**Optimal dimensions:** 240×240 pixels (1:1 ratio)

**Requirements:**
- Minimum size: 120×120 pixels
- Maximum size: 4096×4096 pixels
- Maximum file size: 5MB
- Square aspect ratio (1:1)

**Use case:** When small preview is preferred (profile pages, simple announcements)

### App Card

**Icon dimensions:** 200×200 pixels (1:1 ratio)

**Requirements:**
- Square icon
- Maximum file size: 5MB
- PNG format recommended

### Player Card

**Video thumbnail:** 1200×628 pixels (1.91:1 ratio)

**Requirements:**
- Same as summary_large_image
- Video player must be HTTPS
- Must support Twitter's Player Card API

## Design Best Practices

### Visual Guidelines

**✅ DO:**
- Use high-contrast colors
- Include readable text (minimum 60px font on 1200×628 image)
- Feature primary subject in center
- Add brand logo (small, corner placement)
- Use professional photography/graphics
- Ensure mobile readability
- Test at display size (506×266)
- Include visual hierarchy
- Use consistent brand colors
- Save as JPG for photos (smaller file size)
- Save as PNG for graphics/text (crisp edges)

**❌ DON'T:**
- Small, illegible text
- Complex graphics that don't scale
- Too much text (< 20% of image area)
- Low resolution images
- Clickbait imagery
- Text too close to edges (leave 20px margin)
- White backgrounds (blend with Twitter UI)
- Rely only on color for meaning

### Text Overlay

**Readable text on images:**

```
Font size: 60-120px for headlines (on 1200×628 image)
Font weight: Bold or semi-bold
Background: Semi-transparent overlay for contrast
Contrast ratio: Minimum 4.5:1 (WCAG AA)
Safe area: 40px margin from all edges
```

**Example:**
```css
/* Text overlay design */
.twitter-card-text {
  font-size: 72px;
  font-weight: bold;
  color: #ffffff;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.8);
  /* Or use background overlay */
  background: rgba(0, 0, 0, 0.6);
  padding: 20px;
}
```

### Brand Consistency

**Logo placement:**
- Bottom right or top left corner
- 80-120px width (on 1200×628 image)
- Semi-transparent if needed
- Consistent position across all cards

**Color palette:**
- Use brand colors consistently
- High contrast for readability
- Avoid pure white backgrounds

### Mobile Optimization

**Remember:** Most Twitter users are on mobile

**Mobile considerations:**
- Text must be readable at 345×181 pixels (mobile display)
- Critical information in center
- Test on actual mobile device
- Avoid fine details that disappear when scaled

## File Size Optimization

### Target File Size

**Goal:** < 500KB

**Why:**
- Faster loading on mobile
- Better user experience
- Lower bandwidth costs
- Twitter caching efficiency

### Optimization Techniques

**ImageMagick:**
```bash
# Resize and optimize for Twitter Card
convert input.jpg \
  -resize 1200x628^ \
  -gravity center \
  -extent 1200x628 \
  -quality 85 \
  -strip \
  twitter-card.jpg

# Result: ~200-400KB
```

**Hugo Image Processing:**
```go-html-template
{{ $image := resources.Get "images/hero.jpg" }}
{{ $twitter := $image.Fill "1200x628 center jpg q85" }}
<meta name="twitter:image" content="{{ $twitter.Permalink }}">
```

**WebP (smaller file size):**
```bash
# Convert to WebP (50-80% smaller than JPG)
convert input.jpg -resize 1200x628^ -gravity center -extent 1200x628 -quality 85 twitter-card.webp
```

**Note:** Twitter supports WebP, but JPG is more universally compatible.

**Optimization checklist:**
- ✅ Resize to exact dimensions (1200×628)
- ✅ Use quality 85 (JPG) - sweet spot for size vs quality
- ✅ Strip metadata (EXIF data)
- ✅ Compress with tools (ImageOptim, TinyPNG)
- ✅ Test file size (< 500KB)

## Hugo Implementation

### Basic Template

**layouts/partials/head/twitter-card.html:**

```go-html-template
{{/* Twitter Card */}}
<meta name="twitter:card" content="summary_large_image">

{{ with .Site.Params.twitter_site }}
  <meta name="twitter:site" content="@{{ . }}">
{{ end }}

{{ with .Params.twitter_creator }}
  <meta name="twitter:creator" content="@{{ . }}">
{{ else }}
  {{ with .Site.Params.twitter_creator }}
    <meta name="twitter:creator" content="@{{ . }}">
  {{ end }}
{{ end }}

<meta name="twitter:title" content="{{ .Title }}">
<meta name="twitter:description" content="{{ with .Description }}{{ . }}{{ else }}{{ .Site.Params.description }}{{ end }}">

{{ with .Params.images }}
  {{ range first 1 . }}
    <meta name="twitter:image" content="{{ . | absURL }}">
  {{ end }}
{{ else }}
  {{ with $.Site.Params.twitter_image }}
    <meta name="twitter:image" content="{{ . | absURL }}">
  {{ end }}
{{ end }}

{{ with .Params.twitter_image_alt }}
  <meta name="twitter:image:alt" content="{{ . }}">
{{ end }}
```

### Include in Head

**layouts/_default/baseof.html:**

```go-html-template
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{{ .Title }}</title>

  {{/* SEO meta tags */}}
  {{ partial "head/opengraph.html" . }}
  {{ partial "head/twitter-card.html" . }}

  {{/* Other head elements */}}
</head>
```

### Content Front Matter

**content/blog/hugo-performance-tips.md:**

```yaml
---
title: "10 Essential Hugo Performance Tips"
description: "Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples."
date: 2026-02-13T10:00:00Z
images:
  - /images/blog/hugo-performance-twitter.jpg
twitter_image_alt: "Dashboard showing Hugo site performance improvements with metrics"
twitter_creator: authorhandle
---
```

### Dynamic Image Generation

**Hugo Pipes - Generate Twitter Card:**

```go-html-template
{{ $image := resources.Get "images/blog-hero.jpg" }}
{{ $twitter := $image.Fill "1200x628 center jpg q85" }}

<meta name="twitter:image" content="{{ $twitter.Permalink }}">
<meta name="twitter:image:alt" content="{{ .Title }} - {{ .Site.Title }}">
```

### Automatic Alt Text

```go-html-template
{{ with .Params.images }}
  {{ range first 1 . }}
    <meta name="twitter:image" content="{{ . | absURL }}">
    <meta name="twitter:image:alt" content="{{ $.Title }} - {{ $.Site.Title }}">
  {{ end }}
{{ end }}
```

### Fallback Image

**Site-wide default Twitter Card image:**

**config.toml:**
```toml
[params]
  twitter_site = "yourbrand"
  twitter_creator = "defaultauthor"
  twitter_image = "/images/default-twitter-card.jpg"
```

**Template with fallback:**
```go-html-template
{{ $twitterImage := "" }}

{{/* Check page-specific image */}}
{{ with .Params.images }}
  {{ $twitterImage = index . 0 }}
{{ else }}
  {{/* Check page bundle image */}}
  {{ $image := .Resources.GetMatch "featured-*" }}
  {{ with $image }}
    {{ $twitter := .Fill "1200x628 center jpg q85" }}
    {{ $twitterImage = $twitter.Permalink }}
  {{ else }}
    {{/* Fallback to site default */}}
    {{ $twitterImage = .Site.Params.twitter_image | absURL }}
  {{ end }}
{{ end }}

<meta name="twitter:image" content="{{ $twitterImage }}">
```

## Complete Example

**Full Twitter Card implementation:**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <title>10 Essential Hugo Performance Tips | Hugo Best Practices</title>
  <meta name="description" content="Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples.">

  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:site" content="@hugobest">
  <meta name="twitter:creator" content="@janedoe">
  <meta name="twitter:title" content="10 Essential Hugo Performance Tips">
  <meta name="twitter:description" content="Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples.">
  <meta name="twitter:image" content="https://example.com/images/blog/hugo-performance-twitter.jpg">
  <meta name="twitter:image:alt" content="Dashboard showing Hugo site performance improvements with before/after metrics">

  <!-- Open Graph (fallback for other platforms) -->
  <meta property="og:type" content="article">
  <meta property="og:url" content="https://example.com/blog/hugo-performance-tips/">
  <meta property="og:title" content="10 Essential Hugo Performance Tips">
  <meta property="og:description" content="Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples.">
  <meta property="og:image" content="https://example.com/images/blog/hugo-performance-og.jpg">
  <meta property="og:site_name" content="Hugo Best Practices">
</head>
<body>
  <!-- Content -->
</body>
</html>
```

## Testing Twitter Cards

### Twitter Card Validator

**URL:** https://cards-dev.twitter.com/validator

**How to use:**
1. Enter page URL
2. Click "Preview card"
3. View how Twitter renders your card
4. Check for errors/warnings

**Note:** Requires Twitter developer account (free)

**Alternative:** Post URL as private tweet (only you can see) and check preview

### Manual Testing

**Test in real Twitter environment:**

```bash
# 1. Deploy your page with Twitter Card tags
# 2. Tweet the URL (as private/draft tweet)
# 3. View preview in tweet composer
# 4. Delete draft tweet
```

**What to check:**
- ✅ Image displays correctly
- ✅ Title shows properly (not truncated)
- ✅ Description appears
- ✅ Attribution (@username) displays
- ✅ Image is crisp and readable
- ✅ Mobile preview looks good

### Validation Checklist

```bash
# Check Twitter Card tags exist
curl -s https://example.com/blog/post/ | grep "twitter:"

# Verify image is accessible
curl -I https://example.com/images/twitter-card.jpg

# Check image dimensions
identify https://example.com/images/twitter-card.jpg
# Output: twitter-card.jpg JPEG 1200x628 1200x628+0+0 8-bit sRGB

# Check file size
curl -sI https://example.com/images/twitter-card.jpg | grep -i content-length
# Should be < 500KB (500000 bytes)
```

## Common Issues

### Image Not Showing

**Problem:** Twitter Card image not displaying

**Solutions:**
1. **Use absolute URL:**
   ```html
   <!-- ❌ BAD -->
   <meta name="twitter:image" content="/images/card.jpg">

   <!-- ✅ GOOD -->
   <meta name="twitter:image" content="https://example.com/images/card.jpg">
   ```

2. **Ensure image is publicly accessible:**
   - No authentication required
   - No robots.txt blocking
   - HTTPS preferred

3. **Check image dimensions:**
   - Minimum: 300×157 pixels
   - Recommended: 1200×628 pixels

4. **Verify file size:**
   - Maximum: 5MB
   - Recommended: < 500KB

5. **Clear Twitter cache:**
   - Use Twitter Card Validator
   - Click "Preview card" to refresh cache

### Wrong Image Showing

**Problem:** Old image displayed after update

**Solution:**

```html
<!-- Add cache-busting parameter -->
<meta name="twitter:image" content="https://example.com/images/card.jpg?v=2">

<!-- Or use Twitter Card Validator to clear cache -->
```

### Image Appears Blurry

**Problem:** Image looks pixelated on Twitter

**Solution:**

```bash
# Ensure image is at least 1200×628
convert input.jpg -resize 1200x628^ -gravity center -extent 1200x628 -quality 90 output.jpg

# Use higher quality setting (90 instead of 85)
```

### Title/Description Truncated

**Problem:** Text cut off in preview

**Solution:**

```html
<!-- Keep title under 70 characters -->
<meta name="twitter:title" content="Hugo Performance: 10 Essential Tips">

<!-- Keep description under 200 characters -->
<meta name="twitter:description" content="Learn 10 proven techniques to make your Hugo site load 3x faster.">
```

### Card Type Not Respected

**Problem:** Shows summary card instead of summary_large_image

**Solution:**

```html
<!-- Ensure card type is specified FIRST -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="...">

<!-- Check image meets minimum size: 300×157 -->
```

## Twitter vs Open Graph

### Fallback Behavior

**Twitter uses Open Graph tags as fallback** if Twitter Card tags are missing.

**Mapping:**

| Twitter Card | Open Graph Fallback |
|--------------|---------------------|
| twitter:title | og:title |
| twitter:description | og:description |
| twitter:image | og:image |
| twitter:url | og:url |

**Best practice:** Include both Twitter Card and Open Graph tags.

**Why:**
- Twitter reads both (Twitter tags take precedence)
- Other platforms (Facebook, LinkedIn) use Open Graph
- Redundancy ensures compatibility

### Minimal Implementation (with fallback)

```html
<!-- Open Graph (used by Facebook, LinkedIn, Twitter as fallback) -->
<meta property="og:type" content="article">
<meta property="og:url" content="https://example.com/page/">
<meta property="og:title" content="Page Title">
<meta property="og:description" content="Page description">
<meta property="og:image" content="https://example.com/images/og-image.jpg">

<!-- Twitter Card (overrides OG tags for Twitter) -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@username">
```

**Twitter will use:**
- twitter:card (summary_large_image)
- og:title (as twitter:title fallback)
- og:description (as twitter:description fallback)
- og:image (as twitter:image fallback)

### When to Use Separate Images

**Use different images for Twitter vs Open Graph when:**

- Twitter needs 1.91:1 ratio, Open Graph uses different ratio
- Platform-specific branding
- Different text overlays

**Example:**

```html
<!-- Open Graph: 1200×630 (1.91:1) -->
<meta property="og:image" content="https://example.com/images/og-image.jpg">

<!-- Twitter: Same size, but different design -->
<meta name="twitter:image" content="https://example.com/images/twitter-image.jpg">
```

**When to use same image:**
- Save time and effort
- Consistent branding
- 1200×628 works well on both platforms

## Design Templates

### Image Composition Formula

**Proven layout for Twitter Card images (1200×628):**

```
┌─────────────────────────────────────┐
│  Logo (80px)              [Top/Left]│
│                                     │
│        Main Headline (120px)        │
│                                     │
│         Subheading (60px)           │
│                                     │
│    [Visual element/photo/graphic]   │
│                                     │
│                   Tag/CTA (40px)    │
└─────────────────────────────────────┘
```

**Safe margins:**
- Top/Bottom: 40px
- Left/Right: 60px

### Typography Scale

**For 1200×628 image:**

| Element | Font Size | Weight | Use Case |
|---------|-----------|--------|----------|
| Main headline | 100-140px | Bold | Primary message |
| Subheading | 50-70px | Semi-bold | Secondary info |
| Body text | 35-45px | Regular | Supporting details |
| Tag/CTA | 30-40px | Bold | Call-to-action |

### Color Schemes

**High-contrast combinations:**

| Background | Text | Use Case |
|------------|------|----------|
| Dark blue (#1a1a2e) | White | Professional |
| Orange (#ff6b35) | White | Energetic |
| Black (#000000) | Yellow (#ffd700) | High contrast |
| Navy (#001f3f) | Cyan (#00d4ff) | Tech/modern |
| Purple (#6a0dad) | White | Creative |

**Avoid:**
- ❌ White background (blends with Twitter UI)
- ❌ Light gray text (poor contrast)
- ❌ Yellow text on white (unreadable)

## Hugo Automation

### Auto-Generate Twitter Card

**Generate Twitter Card image from page content:**

**layouts/partials/head/twitter-card-auto.html:**

```go-html-template
{{ $twitterImage := "" }}

{{/* Check for explicit twitter image */}}
{{ with .Params.twitter_image }}
  {{ $twitterImage = . | absURL }}
{{ else }}
  {{/* Check for page images */}}
  {{ with .Params.images }}
    {{ $img := index . 0 }}
    {{ $resource := resources.Get $img }}
    {{ if $resource }}
      {{ $twitter := $resource.Fill "1200x628 center jpg q85" }}
      {{ $twitterImage = $twitter.Permalink }}
    {{ else }}
      {{ $twitterImage = $img | absURL }}
    {{ end }}
  {{ else }}
    {{/* Check page bundle */}}
    {{ $image := .Resources.GetMatch "{twitter,featured,hero}*" }}
    {{ with $image }}
      {{ $twitter := .Fill "1200x628 center jpg q85" }}
      {{ $twitterImage = $twitter.Permalink }}
    {{ else }}
      {{/* Fallback to site default */}}
      {{ with $.Site.Params.twitter_image }}
        {{ $twitterImage = . | absURL }}
      {{ end }}
    {{ end }}
  {{ end }}
{{ end }}

{{ with $twitterImage }}
  <meta name="twitter:image" content="{{ . }}">
  <meta name="twitter:image:alt" content="{{ $.Params.twitter_image_alt | default $.Title }}">
{{ end }}
```

### Category-Specific Defaults

**Use different default images by content section:**

```go-html-template
{{ $defaultImage := "" }}

{{ if eq .Section "blog" }}
  {{ $defaultImage = "/images/twitter/blog-default.jpg" }}
{{ else if eq .Section "tutorials" }}
  {{ $defaultImage = "/images/twitter/tutorial-default.jpg" }}
{{ else if eq .Section "news" }}
  {{ $defaultImage = "/images/twitter/news-default.jpg" }}
{{ else }}
  {{ $defaultImage = .Site.Params.twitter_image }}
{{ end }}

{{ $twitterImage := .Params.twitter_image | default $defaultImage }}
<meta name="twitter:image" content="{{ $twitterImage | absURL }}">
```

### Programmatic Image Generation

**Generate Twitter Card with text overlay (requires external service):**

```go-html-template
{{/* Using external service like Cloudinary or imgix */}}
{{ $title := .Title }}
{{ $baseImage := .Params.hero_image | default "/images/twitter/base.jpg" }}

{{/* Build image URL with text overlay */}}
{{ $twitterURL := printf "https://res.cloudinary.com/yourcloud/image/upload/c_fit,w_1200,h_628/l_text:Arial_80_bold:%s/fl_layer_apply,g_center/%s"
  ($title | urlize)
  ($baseImage | base64Encode)
}}

<meta name="twitter:image" content="{{ $twitterURL }}">
```

**Note:** This requires external image manipulation service (Cloudinary, imgix, etc.)

## Best Practices

### Image Design

**✅ DO:**
- Use 1200×628 pixels for summary_large_image
- Keep file size under 500KB
- Include readable text (minimum 60px font)
- Add brand logo in corner
- Use high-contrast colors
- Test at mobile size (345×181)
- Ensure HTTPS URL
- Add descriptive alt text
- Use consistent branding
- Center important content (safe area)

**❌ DON'T:**
- Use images smaller than 300×157
- Exceed 5MB file size
- Include small, illegible text
- Use pure white backgrounds
- Rely on fine details
- Place critical info near edges
- Use low contrast colors
- Forget alt text (accessibility)
- Ignore mobile preview
- Use authentication-protected images

### Title and Description

**✅ DO:**
- Keep title under 70 characters
- Keep description under 200 characters
- Front-load important keywords
- Make it compelling and clickable
- Include value proposition
- Match image content

**❌ DON'T:**
- Exceed character limits
- Use clickbait
- Include hashtags (use in tweet text instead)
- Duplicate title in description
- Use all caps
- Include URL (Twitter adds it automatically)

### Technical

**✅ DO:**
- Use absolute URLs (HTTPS)
- Include both Twitter and Open Graph tags
- Test with Twitter Card Validator
- Optimize images (< 500KB)
- Use JPG for photos, PNG for graphics
- Strip metadata (EXIF)
- Include alt text for accessibility
- Set up fallback images

**❌ DON'T:**
- Use relative URLs (/images/card.jpg)
- Use HTTP (use HTTPS)
- Skip validation testing
- Upload unoptimized images (> 1MB)
- Forget about mobile users
- Block images in robots.txt
- Require authentication for images

## Advanced Techniques

### Responsive Images

**Serve different images based on device:**

```go-html-template
{{/* Generate multiple sizes */}}
{{ $image := resources.Get "images/hero.jpg" }}
{{ $twitterLarge := $image.Fill "1200x628 center jpg q85" }}
{{ $twitterSmall := $image.Fill "600x314 center jpg q85" }}

{{/* Twitter uses summary_large_image tag, no responsive support */}}
{{/* But you can optimize for file size */}}
<meta name="twitter:image" content="{{ $twitterSmall.Permalink }}">
```

**Note:** Twitter doesn't support responsive images via HTML tags, but you can use smaller images to reduce file size.

### WebP with Fallback

```go-html-template
{{ $image := resources.Get "images/hero.jpg" }}
{{ $twitterWebP := $image.Fill "1200x628 center webp q85" }}
{{ $twitterJPG := $image.Fill "1200x628 center jpg q85" }}

{{/* Twitter supports WebP, but JPG is more compatible */}}
<meta name="twitter:image" content="{{ $twitterJPG.Permalink }}">
```

### Dynamic Text Overlay

**Use Hugo to generate image URLs with dynamic text (requires external service):**

```go-html-template
{{/* Example using Cloudinary */}}
{{ $title := .Title | replaceRE " " "%20" }}
{{ $imageURL := printf "https://res.cloudinary.com/demo/image/upload/w_1200,h_628,c_fill,q_auto,f_auto/l_text:Arial_72_bold:%s,co_rgb:FFFFFF,g_center/base-template.jpg" $title }}

<meta name="twitter:image" content="{{ $imageURL }}">
```

### Conditional Card Types

**Use summary card for some pages, summary_large_image for others:**

```go-html-template
{{ $cardType := "summary_large_image" }}

{{ if eq .Type "author" }}
  {{ $cardType = "summary" }}
{{ end }}

<meta name="twitter:card" content="{{ $cardType }}">
```

## Guidelines

### Essential

**Required for Twitter Card:**
- twitter:card (summary_large_image)
- twitter:title
- twitter:description
- twitter:image (1200×628, HTTPS, < 500KB)

**Minimum viable implementation:**
```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="Page Title">
<meta name="twitter:description" content="Page description">
<meta name="twitter:image" content="https://example.com/image.jpg">
```

### Recommended

**For better engagement:**
- twitter:site (@username)
- twitter:creator (@username)
- twitter:image:alt (accessibility)
- Open Graph tags (fallback for other platforms)
- Optimized image (< 500KB)
- Readable text on image (60px+ font)
- Brand logo on image

### Advanced

**For maximum control:**
- Different images for Twitter vs Open Graph
- Category-specific default images
- Automatic image generation
- Dynamic text overlays (external service)
- A/B testing different images
- Custom card type per page type

## Performance Checklist

**Before deploying:**

- [ ] Image is exactly 1200×628 pixels
- [ ] File size is under 500KB (ideally < 300KB)
- [ ] Image is accessible via HTTPS
- [ ] No authentication required
- [ ] Title is under 70 characters
- [ ] Description is under 200 characters
- [ ] Alt text is provided
- [ ] Tested with Twitter Card Validator
- [ ] Tested on mobile device
- [ ] Text is readable at 345×181 (mobile display size)
- [ ] Brand logo is visible but not dominant
- [ ] High contrast colors used
- [ ] Important content in center (safe area)
- [ ] Both Twitter and Open Graph tags included
- [ ] Fallback image configured

## Benefits

Better Engagement. Rich previews increase click-through rates by 2-3x.

Brand Consistency. Professional appearance across all shared links.

Mobile Optimization. Most Twitter users are on mobile; optimized images ensure readability.

Accessibility. Alt text makes content accessible to visually impaired users.

Universal Compatibility. Works across Twitter, other platforms via Open Graph fallback.

## Related

- [open-graph-meta-tags.md](./open-graph-meta-tags.md) - Open Graph Protocol basics
- [twitter-card-types.md](./twitter-card-types.md) - All Twitter Card types (summary, app, player)
- [social-media-image-generation.md](./social-media-image-generation.md) - Automated image generation
- [opengraph-article-extensions.md](./opengraph-article-extensions.md) - Article-specific metadata
- [meta-descriptions-titles.md](./meta-descriptions-titles.md) - Title and description optimization
