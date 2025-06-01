# Twitter Cards

Twitter link previews. Rich media cards. Summary, summary with large image, player, app cards. X (Twitter) sharing optimization.

## Principle

Control how your content appears when shared on Twitter/X. Provide rich previews with images and metadata. Improve engagement and click-through rates from tweets.

## What are Twitter Cards?

**Purpose:** Metadata that controls how URLs are displayed when shared on Twitter (now called X).

**Format:** `<meta>` tags in HTML `<head>` with `name="twitter:*"` attributes

**Card Types:**
- Summary Card (small image)
- Summary Card with Large Image (featured image)
- Player Card (video/audio)
- App Card (mobile app installs)

**Documentation:** https://developer.twitter.com/en/docs/twitter-for-websites/cards/overview/abouts-cards

**Fallback:** Twitter uses Open Graph tags if Twitter Card tags are missing.

## Basic Twitter Card (Summary)

```html
<!-- Required Twitter Card tags -->
<meta name="twitter:card" content="summary">
<meta name="twitter:title" content="Your Page Title">
<meta name="twitter:description" content="Page description here">
<meta name="twitter:image" content="https://example.com/image.jpg">
```

**Small square image preview** (120×120 displayed, 1:1 aspect ratio).

## Summary Card with Large Image (Recommended)

```html
<!-- Large image card (most common) -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="Complete Guide to Twitter Cards">
<meta name="twitter:description" content="Learn how to implement Twitter Cards for rich link previews on X (Twitter).">
<meta name="twitter:image" content="https://example.com/twitter-image.jpg">
```

**Large image preview** (displayed as 2:1 aspect ratio in feed).

**Recommended:** Use `summary_large_image` for better visibility.

## Complete Twitter Card Tags

```html
<!-- Card type -->
<meta name="twitter:card" content="summary_large_image">

<!-- Content -->
<meta name="twitter:title" content="Complete Guide to Twitter Cards">
<meta name="twitter:description" content="Learn how to implement Twitter Cards for rich link previews on X (Twitter) with images and metadata.">
<meta name="twitter:image" content="https://example.com/twitter-card-image.jpg">
<meta name="twitter:image:alt" content="Twitter Cards guide with example preview">

<!-- Attribution -->
<meta name="twitter:site" content="@example">
<meta name="twitter:creator" content="@janedoe">

<!-- Domain -->
<meta name="twitter:domain" content="example.com">
```

## Card Types

### 1. Summary Card

```html
<meta name="twitter:card" content="summary">
<meta name="twitter:title" content="Page Title">
<meta name="twitter:description" content="Description">
<meta name="twitter:image" content="https://example.com/image.jpg">
```

**Image specs:**
- Minimum: 200×200 pixels
- Recommended: 400×400 pixels
- Aspect ratio: 1:1 (square)
- Max file size: 5 MB

**Use when:** Small preview is appropriate (e.g., profile, about page).

### 2. Summary Card with Large Image

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="Page Title">
<meta name="twitter:description" content="Description">
<meta name="twitter:image" content="https://example.com/image.jpg">
```

**Image specs:**
- Minimum: 300×157 pixels
- Recommended: 1200×628 pixels (same as Open Graph)
- Aspect ratio: 2:1
- Max file size: 5 MB

**Use when:** Showcasing content (articles, blog posts, products).

**Most popular:** This is the most commonly used card type.

### 3. Player Card

```html
<meta name="twitter:card" content="player">
<meta name="twitter:title" content="Video Title">
<meta name="twitter:description" content="Video description">
<meta name="twitter:image" content="https://example.com/video-poster.jpg">
<meta name="twitter:player" content="https://example.com/player.html">
<meta name="twitter:player:width" content="1280">
<meta name="twitter:player:height" content="720">
<meta name="twitter:player:stream" content="https://example.com/video.mp4">
```

**Use when:** Embedding video or audio players.

**Requires:** Twitter approval (whitelist) for player cards.

### 4. App Card

```html
<meta name="twitter:card" content="app">
<meta name="twitter:title" content="App Name">
<meta name="twitter:description" content="App description">
<meta name="twitter:image" content="https://example.com/app-icon.jpg">

<!-- iPhone -->
<meta name="twitter:app:name:iphone" content="App Name">
<meta name="twitter:app:id:iphone" content="1234567890">
<meta name="twitter:app:url:iphone" content="app://open">

<!-- iPad -->
<meta name="twitter:app:name:ipad" content="App Name">
<meta name="twitter:app:id:ipad" content="1234567890">

<!-- Android -->
<meta name="twitter:app:name:googleplay" content="App Name">
<meta name="twitter:app:id:googleplay" content="com.example.app">
<meta name="twitter:app:url:googleplay" content="app://open">
```

**Use when:** Promoting mobile app installs.

## Image Specifications

### Summary Card (Small Image)

**Displayed size:** 120×120 pixels (cropped to square)

**Upload specs:**
- Minimum: 200×200
- Recommended: 400×400
- Aspect ratio: 1:1 (square)
- Format: JPG, PNG, WebP
- Max size: 5 MB

### Summary Large Image Card

**Displayed size:** 2:1 aspect ratio in timeline

**Upload specs:**
- Minimum: 300×157
- Recommended: 1200×628 (same as Open Graph)
- Aspect ratio: 2:1
- Format: JPG, PNG, WebP
- Max size: 5 MB

**Design tips:**
- Important content in center (safe area)
- Avoid text smaller than 60px
- High contrast
- Test on mobile (most Twitter users are mobile)

## Hugo Implementation

### Hugo Template (Head Partial)

```go-html-template
{{/* layouts/partials/head.html */}}

<!-- Twitter Card tags -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="{{ .Title }}">

{{- with .Description }}
<meta name="twitter:description" content="{{ . }}">
{{- else }}
<meta name="twitter:description" content="{{ .Site.Params.description }}">
{{- end }}

{{- with .Params.image }}
<meta name="twitter:image" content="{{ . | absURL }}">
{{- else }}
<meta name="twitter:image" content="{{ "/images/twitter-default.jpg" | absURL }}">
{{- end }}

{{- with .Params.imageAlt }}
<meta name="twitter:image:alt" content="{{ . }}">
{{- else }}
<meta name="twitter:image:alt" content="{{ .Title }}">
{{- end }}

{{- with .Site.Params.twitter }}
<meta name="twitter:site" content="@{{ . }}">
{{- end }}

{{- with .Params.author }}
{{- with $.Site.Params.authors }}
{{- with index . $author }}
{{- with .twitter }}
<meta name="twitter:creator" content="@{{ . }}">
{{- end }}
{{- end }}
{{- end }}
{{- end }}
```

### Configuration (config.toml)

```toml
[params]
twitter = "example"  # Your Twitter handle (without @)
description = "Default site description"

[params.authors.janedoe]
name = "Jane Doe"
twitter = "janedoe"
```

### Content Front Matter

```yaml
---
title: "Complete Guide to Twitter Cards"
date: 2026-02-13
description: "Learn how to implement Twitter Cards for rich link previews."
image: "/images/posts/twitter-cards-guide.jpg"
imageAlt: "Twitter Cards guide diagram"
author: "janedoe"
---
```

## Twitter Card + Open Graph (Combined)

**Best practice:** Use both Twitter Cards and Open Graph tags.

```html
<!-- Open Graph (Facebook, LinkedIn, etc.) -->
<meta property="og:title" content="Page Title">
<meta property="og:description" content="Description">
<meta property="og:image" content="https://example.com/og-image.jpg">
<meta property="og:url" content="https://example.com/page">
<meta property="og:type" content="article">

<!-- Twitter Card -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="Page Title">
<meta name="twitter:description" content="Description">
<meta name="twitter:image" content="https://example.com/twitter-image.jpg">
```

**Twitter fallback:** If `twitter:title` is missing, Twitter uses `og:title`.

**Minimal approach:**

```html
<!-- Open Graph (primary) -->
<meta property="og:title" content="Page Title">
<meta property="og:description" content="Description">
<meta property="og:image" content="https://example.com/image.jpg">

<!-- Twitter Card (just card type) -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@example">

<!-- Twitter will fallback to OG tags for title, description, image -->
```

## Hugo Template (Combined OG + Twitter)

```go-html-template
{{/* layouts/partials/meta.html */}}

{{- $title := .Title }}
{{- $description := .Description | default .Site.Params.description }}
{{- $image := .Params.image | default "/images/default-card.jpg" | absURL }}

<!-- Open Graph -->
<meta property="og:title" content="{{ $title }}">
<meta property="og:description" content="{{ $description }}">
<meta property="og:image" content="{{ $image }}">
<meta property="og:url" content="{{ .Permalink }}">
<meta property="og:type" content="{{ if .IsPage }}article{{ else }}website{{ end }}">
<meta property="og:site_name" content="{{ .Site.Title }}">

<!-- Twitter Card -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="{{ $title }}">
<meta name="twitter:description" content="{{ $description }}">
<meta name="twitter:image" content="{{ $image }}">

{{- with .Params.imageAlt }}
<meta name="twitter:image:alt" content="{{ . }}">
{{- end }}

{{- with .Site.Params.twitter }}
<meta name="twitter:site" content="@{{ . }}">
{{- end }}

{{- with .Params.twitterCreator }}
<meta name="twitter:creator" content="@{{ . }}">
{{- end }}
```

## Testing Twitter Cards

### Twitter Card Validator

**URL:** https://cards-dev.twitter.com/validator

**How to use:**
1. Enter your URL
2. Click "Preview card"
3. Review the preview
4. Check for errors

**Note:** As of 2024, Twitter/X may have changed validator URLs. Search for "Twitter Card Validator" if the link doesn't work.

### Manual Testing

```bash
# Check if Twitter Card tags exist
curl https://example.com | grep "twitter:"

# Extract Twitter Card tags
curl https://example.com | grep -o '<meta name="twitter:[^"]*" content="[^"]*"'

# Validate image URL
curl -I https://example.com/twitter-image.jpg
# Should return 200 OK
```

### Test by Sharing

**Best method:** Actually share the URL on Twitter/X:

1. Create a test tweet with your URL
2. View the preview
3. Delete the tweet if needed
4. Check on mobile and desktop

## Player Card (Video/Audio)

```html
<!-- Player card for embedded video -->
<meta name="twitter:card" content="player">
<meta name="twitter:title" content="Video Title">
<meta name="twitter:description" content="Video description">
<meta name="twitter:image" content="https://example.com/video-poster.jpg">

<!-- Player embed URL -->
<meta name="twitter:player" content="https://example.com/player.html">
<meta name="twitter:player:width" content="1280">
<meta name="twitter:player:height" content="720">

<!-- Direct stream (optional) -->
<meta name="twitter:player:stream" content="https://example.com/video.mp4">
<meta name="twitter:player:stream:content_type" content="video/mp4">
```

**Player iframe (player.html):**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Video Player</title>
</head>
<body style="margin:0;padding:0;">
  <video width="100%" height="100%" controls autoplay>
    <source src="https://example.com/video.mp4" type="video/mp4">
  </video>
</body>
</html>
```

**Note:** Player cards require approval from Twitter. Most sites use summary_large_image instead.

## Attribution Tags

```html
<!-- Site attribution (your Twitter account) -->
<meta name="twitter:site" content="@example">

<!-- Content creator attribution (author's Twitter account) -->
<meta name="twitter:creator" content="@janedoe">

<!-- Domain -->
<meta name="twitter:domain" content="example.com">
```

**Hugo template:**

```go-html-template
{{- with .Site.Params.twitter }}
<meta name="twitter:site" content="@{{ . }}">
{{- end }}

{{- if .Params.author }}
  {{- with index .Site.Params.authors .Params.author }}
    {{- with .twitter }}
<meta name="twitter:creator" content="@{{ . }}">
    {{- end }}
  {{- end }}
{{- end }}
```

## Common Mistakes

❌ **Relative URLs for images:**
```html
<!-- Wrong -->
<meta name="twitter:image" content="/images/card.jpg">

<!-- Correct (absolute URL required) -->
<meta name="twitter:image" content="https://example.com/images/card.jpg">
```

❌ **Wrong image dimensions:**
```html
<!-- Wrong (too small for large image card) -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:image" content="https://example.com/small-logo.png">
<!-- Image is 200x200, needs to be ~1200x628 -->

<!-- Correct -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:image" content="https://example.com/twitter-card.jpg">
<!-- Image is 1200x628 -->
```

❌ **Using `property` instead of `name`:**
```html
<!-- Wrong -->
<meta property="twitter:card" content="summary_large_image">

<!-- Correct -->
<meta name="twitter:card" content="summary_large_image">
```

❌ **Including @ in handle:**
```html
<!-- Wrong -->
<meta name="twitter:site" content="@@example">

<!-- Correct -->
<meta name="twitter:site" content="@example">
```

❌ **Not providing image alt text:**
```html
<!-- Better (but missing alt) -->
<meta name="twitter:image" content="https://example.com/image.jpg">

<!-- Best (with alt text for accessibility) -->
<meta name="twitter:image" content="https://example.com/image.jpg">
<meta name="twitter:image:alt" content="Descriptive alt text">
```

## Dynamic Twitter Card Images

### Option 1: Use OG Image Services

```html
<!-- Vercel OG Image -->
<meta name="twitter:image" content="https://og-image.vercel.app/{{ .Title }}.png?theme=light&md=1">
```

### Option 2: Hugo Shortcode

```go-html-template
{{/* layouts/shortcodes/twitter-card.html */}}

{{- $title := .Get "title" | default $.Page.Title }}
{{- $description := .Get "description" | default $.Page.Description }}

{{/* Generate dynamic image URL */}}
{{- $imageURL := printf "https://og-image.example.com/?title=%s&desc=%s"
    (urlquery $title)
    (urlquery $description) }}

<meta name="twitter:image" content="{{ $imageURL }}">
```

### Option 3: Cloudinary Transform

```go-html-template
{{- $title := .Title }}
{{- $cloudinaryURL := printf "https://res.cloudinary.com/demo/image/upload/l_text:Arial_80:%s/twitter-template.jpg" (urlquery $title) }}

<meta name="twitter:image" content="{{ $cloudinaryURL }}">
```

## Analytics and Tracking

### Twitter Analytics

**Enable:** Twitter provides analytics for shared links if you have a Twitter account.

**View:** Twitter Analytics Dashboard shows:
- Impressions
- Engagements
- Link clicks
- Retweets

### UTM Parameters

```html
<!-- Add UTM parameters to track Twitter traffic -->
<meta name="twitter:url" content="https://example.com/page?utm_source=twitter&utm_medium=social&utm_campaign=twitter-card">
```

**Hugo template:**

```go-html-template
{{- $baseURL := .Permalink }}
{{- $utmURL := printf "%s?utm_source=twitter&utm_medium=social" $baseURL }}

<meta name="twitter:url" content="{{ $utmURL }}">
```

## Best Practices

**Required:**
- `twitter:card` (card type)
- `twitter:title` (or fallback to `og:title`)
- `twitter:image` (or fallback to `og:image`)

**Recommended:**
- Use `summary_large_image` (better engagement)
- Include `twitter:description`
- Add `twitter:image:alt` (accessibility)
- Set `twitter:site` (attribution)
- Absolute URLs for images (https://)

**Image Guidelines:**
- 1200×628 pixels for large image cards
- < 5 MB file size
- JPG or PNG format
- High quality, clear visuals

**Testing:**
- Use Twitter Card Validator
- Test actual sharing on Twitter/X
- Check mobile and desktop
- Verify images load correctly

## Hugo Complete Example

```go-html-template
{{/* layouts/partials/twitter-cards.html */}}

{{- $title := .Title }}
{{- $description := .Description | default .Site.Params.description }}
{{- $image := "" }}

{{- if .Params.twitterImage }}
  {{- $image = .Params.twitterImage | absURL }}
{{- else if .Params.image }}
  {{- $image = .Params.image | absURL }}
{{- else }}
  {{- $image = "/images/twitter-default.jpg" | absURL }}
{{- end }}

<!-- Twitter Card -->
<meta name="twitter:card" content="{{ .Params.twitterCard | default "summary_large_image" }}">
<meta name="twitter:title" content="{{ $title }}">
<meta name="twitter:description" content="{{ $description }}">
<meta name="twitter:image" content="{{ $image }}">

{{- with .Params.imageAlt }}
<meta name="twitter:image:alt" content="{{ . }}">
{{- else }}
<meta name="twitter:image:alt" content="{{ $title }}">
{{- end }}

{{- with .Site.Params.twitter }}
<meta name="twitter:site" content="@{{ . }}">
{{- end }}

{{- with .Params.twitterCreator }}
<meta name="twitter:creator" content="@{{ . }}">
{{- else if .Params.author }}
  {{- with index .Site.Params.authors .Params.author }}
    {{- with .twitter }}
<meta name="twitter:creator" content="@{{ . }}">
    {{- end }}
  {{- end }}
{{- end }}
```

## Guidelines

**Essential:**
```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="...">
<meta name="twitter:image" content="https://...">
```

**Recommended:**
```html
<meta name="twitter:description" content="...">
<meta name="twitter:image:alt" content="...">
<meta name="twitter:site" content="@...">
```

**Image Requirements:**
- 1200×628 pixels (2:1 ratio)
- Absolute URL (https://)
- JPG or PNG format
- < 5 MB file size

## Benefits

Shareable. Rich previews on Twitter/X.

Visual. Large images attract attention.

Branded. Include your Twitter handle.

Measurable. Track engagement analytics.

## Related

- [open-graph-protocol.md](./open-graph-protocol.md) - Open Graph tags
- [meta-tags-seo.md](./meta-tags-seo.md) - HTML meta tags
- [structured-data-json-ld.md](./structured-data-json-ld.md) - Structured data
