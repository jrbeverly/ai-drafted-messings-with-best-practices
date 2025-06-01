# Twitter Card Types

Twitter Card variations. Summary cards. Large image cards. App cards. Player cards. Rich media previews.

## Principle

Use the appropriate Twitter Card type for your content. Match card type to content format. Optimize engagement by choosing the right preview style for your use case.

## What are Twitter Card Types?

**Twitter Cards:** Rich media previews for links shared on Twitter/X

**Card Types:**
1. **Summary** - Small square image with title and description
2. **Summary Large Image** - Large rectangular image (most popular)
3. **App** - Mobile app install cards
4. **Player** - Video/audio player embedded in tweet

**Purpose:** Enhanced link previews that increase engagement and clicks

**Specification:** https://developer.twitter.com/en/docs/twitter-for-websites/cards/overview/abouts-cards

## Summary Card

**Small image preview with title and description.**

### When to Use

**Best for:**
- Profile pages
- Articles without featured images
- Text-focused content
- App profiles
- Company pages
- Author bios

**Not ideal for:**
- Blog posts with hero images (use summary_large_image instead)
- Visual content (photography, design portfolios)
- Video/audio content (use player card)

### Specifications

**Image:**
- Dimensions: 240×240 pixels (1:1 ratio)
- Minimum: 120×120 pixels
- Maximum: 4096×4096 pixels
- File size: Maximum 5MB
- Formats: JPG, PNG, WebP, GIF

**Text:**
- Title: Maximum 70 characters
- Description: Maximum 200 characters

**Display size:**
- Desktop: 120×120 pixels (small square)
- Mobile: 120×120 pixels

### Implementation

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta name="twitter:card" content="summary">
  <meta name="twitter:site" content="@yourusername">
  <meta name="twitter:title" content="Jane Doe - Author Profile">
  <meta name="twitter:description" content="Web developer, writer, and Hugo enthusiast. Sharing tips on static sites and web performance.">
  <meta name="twitter:image" content="https://example.com/images/authors/jane-doe-square.jpg">
  <meta name="twitter:image:alt" content="Profile photo of Jane Doe">
</head>
</html>
```

### Hugo Template

**layouts/partials/head/twitter-summary.html:**

```go-html-template
{{/* Summary card (small square image) */}}
<meta name="twitter:card" content="summary">

{{ with .Site.Params.twitter_site }}
  <meta name="twitter:site" content="@{{ . }}">
{{ end }}

<meta name="twitter:title" content="{{ .Title }}">
<meta name="twitter:description" content="{{ with .Description }}{{ . }}{{ else }}{{ .Site.Params.description }}{{ end }}">

{{ $image := "" }}
{{ with .Params.avatar }}
  {{ $image = . }}
{{ else }}
  {{ with .Params.images }}
    {{ $img := index . 0 }}
    {{ $resource := resources.Get $img }}
    {{ if $resource }}
      {{/* Crop to square */}}
      {{ $square := $resource.Fill "240x240 center jpg q85" }}
      {{ $image = $square.Permalink }}
    {{ else }}
      {{ $image = $img | absURL }}
    {{ end }}
  {{ end }}
{{ end }}

{{ with $image }}
  <meta name="twitter:image" content="{{ . }}">
  <meta name="twitter:image:alt" content="{{ $.Params.twitter_image_alt | default $.Title }}">
{{ end }}
```

### Example Use Cases

**Author/Profile Page:**
```yaml
---
title: "Jane Doe - Author"
description: "Web developer and technical writer specializing in Hugo and static sites"
avatar: /images/authors/jane-doe-square.jpg
twitter_card_type: summary
---
```

**Company About Page:**
```yaml
---
title: "About Hugo Best Practices"
description: "Learn about our mission to help developers build faster static sites"
images:
  - /images/company-logo-square.png
twitter_card_type: summary
---
```

## Summary Large Image

**Large rectangular image with title and description (most popular).**

### When to Use

**Best for:**
- Blog posts with featured images
- News articles
- Product pages
- Landing pages
- Visual content
- Tutorials with screenshots
- Photography/design portfolios

**This is the default choice for most content.**

### Specifications

**Image:**
- Dimensions: 1200×628 pixels (1.91:1 ratio)
- Minimum: 300×157 pixels
- Maximum: 4096×4096 pixels
- File size: Maximum 5MB (recommended < 500KB)
- Formats: JPG, PNG, WebP, GIF

**Text:**
- Title: Maximum 70 characters
- Description: Maximum 200 characters

**Display size:**
- Desktop: 506×266 pixels
- Mobile: Full width, proportional height

### Implementation

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:site" content="@hugobest">
  <meta name="twitter:creator" content="@janedoe">
  <meta name="twitter:title" content="10 Essential Hugo Performance Tips">
  <meta name="twitter:description" content="Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples.">
  <meta name="twitter:image" content="https://example.com/images/blog/hugo-performance-twitter.jpg">
  <meta name="twitter:image:alt" content="Dashboard showing Hugo site performance improvements">
</head>
</html>
```

### Hugo Template

**layouts/partials/head/twitter-large-image.html:**

```go-html-template
{{/* Summary large image card (rectangular image) */}}
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

{{ $image := "" }}
{{ with .Params.images }}
  {{ $img := index . 0 }}
  {{ $resource := resources.Get $img }}
  {{ if $resource }}
    {{ $twitter := $resource.Fill "1200x628 center jpg q85" }}
    {{ $image = $twitter.Permalink }}
  {{ else }}
    {{ $image = $img | absURL }}
  {{ end }}
{{ else }}
  {{ with .Site.Params.twitter_image }}
    {{ $image = . | absURL }}
  {{ end }}
{{ end }}

{{ with $image }}
  <meta name="twitter:image" content="{{ . }}">
  <meta name="twitter:image:alt" content="{{ $.Params.twitter_image_alt | default $.Title }}">
{{ end }}
```

### Example Use Cases

**Blog Post:**
```yaml
---
title: "10 Essential Hugo Performance Tips"
description: "Learn 10 proven techniques to make your Hugo site load 3x faster"
date: 2026-02-13
images:
  - /images/blog/hugo-performance-twitter.jpg
twitter_image_alt: "Dashboard showing performance metrics"
twitter_creator: janedoe
---
```

**Product Page:**
```yaml
---
title: "Hugo Pro Theme - Premium Static Site Theme"
description: "Professional Hugo theme with 50+ components and dark mode support"
images:
  - /images/products/hugo-pro-preview.jpg
twitter_card_type: summary_large_image
---
```

## App Card

**Mobile app installation card with app store links.**

### When to Use

**Best for:**
- Mobile app promotion
- App landing pages
- App store links
- Deep links to app content

**Not for:**
- Web applications (use summary_large_image)
- Desktop software
- Non-mobile apps

### Specifications

**Image:**
- Icon: 200×200 pixels (1:1 ratio, square)
- Minimum: 120×120 pixels
- File size: Maximum 5MB
- Formats: PNG recommended (transparency support)

**Text:**
- App name: Pulled from app store
- Description: Maximum 200 characters

**Required:**
- App ID (iOS or Android)
- App store URL

### Implementation

**iOS App:**

```html
<meta name="twitter:card" content="app">
<meta name="twitter:site" content="@yourapp">
<meta name="twitter:description" content="Download our app to track your Hugo builds and deployments">

<!-- iPhone -->
<meta name="twitter:app:name:iphone" content="Hugo Tracker">
<meta name="twitter:app:id:iphone" content="307234931">
<meta name="twitter:app:url:iphone" content="hugotrac://profile/123">

<!-- iPad -->
<meta name="twitter:app:name:ipad" content="Hugo Tracker">
<meta name="twitter:app:id:ipad" content="307234931">
<meta name="twitter:app:url:ipad" content="hugotrac://profile/123">
```

**Android App:**

```html
<meta name="twitter:card" content="app">
<meta name="twitter:site" content="@yourapp">
<meta name="twitter:description" content="Download our app to track your Hugo builds and deployments">

<meta name="twitter:app:name:googleplay" content="Hugo Tracker">
<meta name="twitter:app:id:googleplay" content="com.example.hugotracker">
<meta name="twitter:app:url:googleplay" content="hugotrac://profile/123">
```

**Both iOS and Android:**

```html
<meta name="twitter:card" content="app">
<meta name="twitter:site" content="@yourapp">
<meta name="twitter:description" content="Download our app to track your Hugo builds and deployments">

<!-- iOS -->
<meta name="twitter:app:name:iphone" content="Hugo Tracker">
<meta name="twitter:app:id:iphone" content="307234931">
<meta name="twitter:app:url:iphone" content="hugotrac://profile/123">

<!-- Android -->
<meta name="twitter:app:name:googleplay" content="Hugo Tracker">
<meta name="twitter:app:id:googleplay" content="com.example.hugotracker">
<meta name="twitter:app:url:googleplay" content="hugotrac://profile/123">
```

### Hugo Template

**layouts/partials/head/twitter-app.html:**

```go-html-template
{{/* App card for mobile apps */}}
<meta name="twitter:card" content="app">

{{ with .Site.Params.twitter_site }}
  <meta name="twitter:site" content="@{{ . }}">
{{ end }}

<meta name="twitter:description" content="{{ with .Description }}{{ . }}{{ else }}{{ .Site.Params.description }}{{ end }}">

{{/* iOS App */}}
{{ with .Params.app_ios_name }}
  <meta name="twitter:app:name:iphone" content="{{ . }}">
  <meta name="twitter:app:name:ipad" content="{{ . }}">
{{ end }}

{{ with .Params.app_ios_id }}
  <meta name="twitter:app:id:iphone" content="{{ . }}">
  <meta name="twitter:app:id:ipad" content="{{ . }}">
{{ end }}

{{ with .Params.app_ios_url }}
  <meta name="twitter:app:url:iphone" content="{{ . }}">
  <meta name="twitter:app:url:ipad" content="{{ . }}">
{{ end }}

{{/* Android App */}}
{{ with .Params.app_android_name }}
  <meta name="twitter:app:name:googleplay" content="{{ . }}">
{{ end }}

{{ with .Params.app_android_id }}
  <meta name="twitter:app:id:googleplay" content="{{ . }}">
{{ end }}

{{ with .Params.app_android_url }}
  <meta name="twitter:app:url:googleplay" content="{{ . }}">
{{ end }}
```

### Front Matter

```yaml
---
title: "Hugo Tracker Mobile App"
description: "Download our app to track your Hugo builds and deployments on the go"
app_ios_name: "Hugo Tracker"
app_ios_id: "307234931"
app_ios_url: "hugotrac://home"
app_android_name: "Hugo Tracker"
app_android_id: "com.example.hugotracker"
app_android_url: "hugotrac://home"
---
```

### Deep Linking

**Link directly to content within the app:**

```html
<!-- Link to specific profile in app -->
<meta name="twitter:app:url:iphone" content="hugotrac://profile/janedoe">

<!-- Link to specific post in app -->
<meta name="twitter:app:url:googleplay" content="hugotrac://post/12345">
```

**User experience:**
1. User clicks link in Twitter
2. If app installed → Opens app to specific content
3. If app not installed → Redirects to app store

## Player Card

**Embedded video/audio player in tweet.**

### When to Use

**Best for:**
- Video content
- Audio/podcasts
- Live streams
- Interactive media
- Embedded players

**Not for:**
- Static images
- Text content
- Links to video hosting platforms (YouTube, Vimeo) - they have their own cards

### Specifications

**Image (poster/thumbnail):**
- Dimensions: 1200×628 pixels (1.91:1 ratio)
- Minimum: 300×157 pixels
- File size: Maximum 5MB
- Formats: JPG, PNG, WebP

**Player:**
- HTTPS required
- Must support Twitter's Player Card API
- Recommended size: 360×200 pixels (minimum)
- Maximum size: 1920×1080 pixels

**Text:**
- Title: Maximum 70 characters
- Description: Maximum 200 characters

### Implementation

```html
<meta name="twitter:card" content="player">
<meta name="twitter:site" content="@yoursite">
<meta name="twitter:title" content="Introduction to Hugo Static Sites">
<meta name="twitter:description" content="Learn the basics of Hugo in this 10-minute tutorial video">
<meta name="twitter:image" content="https://example.com/images/video-poster.jpg">
<meta name="twitter:player" content="https://example.com/embed/video123">
<meta name="twitter:player:width" content="1280">
<meta name="twitter:player:height" content="720">
<meta name="twitter:player:stream" content="https://example.com/videos/hugo-intro.mp4">
```

### Required Tags

**twitter:player** - URL to iframe player
```html
<meta name="twitter:player" content="https://example.com/embed/video123">
```

**twitter:player:width** - Player width in pixels
```html
<meta name="twitter:player:width" content="1280">
```

**twitter:player:height** - Player height in pixels
```html
<meta name="twitter:player:height" content="720">
```

### Optional Tags

**twitter:player:stream** - Direct video URL (MP4, M3U8)
```html
<meta name="twitter:player:stream" content="https://example.com/videos/video.mp4">
<meta name="twitter:player:stream:content_type" content="video/mp4">
```

**twitter:image** - Poster image (shown before play)
```html
<meta name="twitter:image" content="https://example.com/images/video-poster.jpg">
```

### Hugo Template

**layouts/partials/head/twitter-player.html:**

```go-html-template
{{/* Player card for video/audio */}}
<meta name="twitter:card" content="player">

{{ with .Site.Params.twitter_site }}
  <meta name="twitter:site" content="@{{ . }}">
{{ end }}

<meta name="twitter:title" content="{{ .Title }}">
<meta name="twitter:description" content="{{ with .Description }}{{ . }}{{ else }}{{ .Site.Params.description }}{{ end }}">

{{/* Poster image */}}
{{ with .Params.video_poster }}
  <meta name="twitter:image" content="{{ . | absURL }}">
{{ else }}
  {{ with .Params.images }}
    <meta name="twitter:image" content="{{ index . 0 | absURL }}">
  {{ end }}
{{ end }}

{{/* Player embed URL */}}
{{ with .Params.video_player_url }}
  <meta name="twitter:player" content="{{ . }}">
{{ end }}

{{/* Player dimensions */}}
{{ with .Params.video_width }}
  <meta name="twitter:player:width" content="{{ . }}">
{{ else }}
  <meta name="twitter:player:width" content="1280">
{{ end }}

{{ with .Params.video_height }}
  <meta name="twitter:player:height" content="{{ . }}">
{{ else }}
  <meta name="twitter:player:height" content="720">
{{ end }}

{{/* Direct video stream (optional) */}}
{{ with .Params.video_stream_url }}
  <meta name="twitter:player:stream" content="{{ . }}">
  <meta name="twitter:player:stream:content_type" content="{{ $.Params.video_content_type | default "video/mp4" }}">
{{ end }}
```

### Front Matter

```yaml
---
title: "Introduction to Hugo Static Sites"
description: "Learn the basics of Hugo in this 10-minute tutorial video"
video_poster: /images/videos/hugo-intro-poster.jpg
video_player_url: https://example.com/embed/hugo-intro
video_stream_url: https://example.com/videos/hugo-intro.mp4
video_content_type: video/mp4
video_width: 1280
video_height: 720
---
```

### Player Requirements

**Your embed player must:**
- ✅ Use HTTPS
- ✅ Be accessible without authentication
- ✅ Support iframe embedding
- ✅ Implement proper CORS headers
- ✅ Work on mobile devices
- ✅ Provide playback controls

**Example embed player HTML:**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    body { margin: 0; padding: 0; background: black; }
    video { width: 100%; height: 100%; }
  </style>
</head>
<body>
  <video controls autoplay>
    <source src="https://example.com/videos/hugo-intro.mp4" type="video/mp4">
  </video>
</body>
</html>
```

### Audio Player

**For podcasts/audio content:**

```html
<meta name="twitter:card" content="player">
<meta name="twitter:title" content="Hugo Best Practices Podcast Episode 5">
<meta name="twitter:description" content="Tips for optimizing Hugo build times">
<meta name="twitter:image" content="https://example.com/images/podcast-cover.jpg">
<meta name="twitter:player" content="https://example.com/embed/audio-player">
<meta name="twitter:player:width" content="480">
<meta name="twitter:player:height" content="200">
<meta name="twitter:player:stream" content="https://example.com/audio/episode5.mp3">
<meta name="twitter:player:stream:content_type" content="audio/mpeg">
```

## Choosing Card Type

### Decision Matrix

**Use this flow chart to choose the right card type:**

```
Is it a mobile app?
├─ YES → app card
└─ NO ↓

Does it have video/audio player?
├─ YES → player card
└─ NO ↓

Does it have a featured image?
├─ YES → summary_large_image
└─ NO ↓

Is it a profile/bio page?
├─ YES → summary
└─ NO → summary_large_image (default)
```

### Content Type Mapping

| Content Type | Recommended Card | Alternative |
|--------------|------------------|-------------|
| Blog post | summary_large_image | summary |
| News article | summary_large_image | - |
| Product page | summary_large_image | - |
| Author bio | summary | - |
| Company about | summary | - |
| Video content | player | summary_large_image |
| Podcast | player | summary |
| Mobile app | app | summary_large_image |
| Landing page | summary_large_image | - |
| Documentation | summary | - |
| Portfolio | summary_large_image | - |

## Dynamic Card Type Selection

### Hugo Conditional Template

**Automatically choose card type based on content:**

**layouts/partials/head/twitter-card.html:**

```go-html-template
{{/* Determine card type based on content */}}
{{ $cardType := "summary_large_image" }}

{{ if .Params.twitter_card_type }}
  {{ $cardType = .Params.twitter_card_type }}
{{ else if eq .Type "author" }}
  {{ $cardType = "summary" }}
{{ else if .Params.video_player_url }}
  {{ $cardType = "player" }}
{{ else if .Params.app_ios_id }}
  {{ $cardType = "app" }}
{{ else if .Params.images }}
  {{ $cardType = "summary_large_image" }}
{{ else }}
  {{ $cardType = "summary" }}
{{ end }}

<meta name="twitter:card" content="{{ $cardType }}">

{{/* Load type-specific template */}}
{{ if eq $cardType "summary" }}
  {{ partial "head/twitter-summary.html" . }}
{{ else if eq $cardType "summary_large_image" }}
  {{ partial "head/twitter-large-image.html" . }}
{{ else if eq $cardType "app" }}
  {{ partial "head/twitter-app.html" . }}
{{ else if eq $cardType "player" }}
  {{ partial "head/twitter-player.html" . }}
{{ end }}
```

### Override in Front Matter

**Allow manual override per page:**

```yaml
---
title: "My Page"
twitter_card_type: summary  # Override default
---
```

## Testing Card Types

### Twitter Card Validator

**URL:** https://cards-dev.twitter.com/validator

**How to test:**
1. Enter page URL
2. Validator shows card preview
3. Check card type is correct
4. Verify all fields display properly

**Test each card type:**
- ✅ Summary card shows square image
- ✅ Summary large image shows rectangular image
- ✅ App card shows install button
- ✅ Player card shows play button

### Manual Testing

**Test in real Twitter:**

```bash
# 1. Deploy page with Twitter Card tags
# 2. Create private tweet with URL
# 3. Verify preview matches expected card type
# 4. Delete draft tweet
```

## Common Issues

### Wrong Card Type Displaying

**Problem:** Summary card shows instead of summary_large_image

**Causes:**
1. Image too small (< 300×157)
2. Card type not specified
3. twitter:card tag missing

**Solution:**

```html
<!-- Ensure card type is FIRST tag -->
<meta name="twitter:card" content="summary_large_image">

<!-- Ensure image meets minimum size -->
<meta name="twitter:image" content="https://example.com/image-1200x628.jpg">
```

### Player Card Not Working

**Problem:** Video doesn't play in Twitter

**Causes:**
1. Player URL not HTTPS
2. CORS headers missing
3. Player not accessible
4. iframe restrictions

**Solution:**

```html
<!-- Ensure HTTPS -->
<meta name="twitter:player" content="https://example.com/embed/video">

<!-- Server must send CORS headers -->
Access-Control-Allow-Origin: https://twitter.com
X-Frame-Options: ALLOW-FROM https://twitter.com
```

### App Card Not Showing Install Button

**Problem:** App card displays but no install button

**Causes:**
1. Missing app ID
2. Invalid app ID
3. App not published in store

**Solution:**

```html
<!-- Verify app ID is correct -->
<meta name="twitter:app:id:iphone" content="307234931">

<!-- Ensure app is published (not in beta) -->
```

### Image Not Loading

**Problem:** Card shows broken image

**Solution:**

```html
<!-- Use absolute URL with HTTPS -->
<meta name="twitter:image" content="https://example.com/image.jpg">

<!-- Ensure image is publicly accessible (no auth required) -->
```

## Best Practices

### General

**✅ DO:**
- Use summary_large_image as default
- Include both Twitter Card and Open Graph tags
- Test with Twitter Card Validator
- Choose card type based on content
- Provide alt text for images
- Use HTTPS for all URLs
- Keep title under 70 characters
- Keep description under 200 characters

**❌ DON'T:**
- Use same card type for all content
- Skip testing
- Use relative URLs
- Forget alt text
- Use HTTP (use HTTPS)
- Exceed character limits
- Use authentication-protected images

### Summary Card

**✅ DO:**
- Use square images (1:1 ratio)
- Minimum 240×240 pixels
- Use for profiles and bios
- Ensure image works at small size

**❌ DON'T:**
- Use for blog posts with hero images
- Use rectangular images
- Include fine details (too small to see)

### Summary Large Image

**✅ DO:**
- Use 1200×628 pixels
- Include readable text (60px+ font)
- Test at mobile size (345×181)
- Optimize file size (< 500KB)
- Use high-contrast colors

**❌ DON'T:**
- Use images smaller than 300×157
- Include small, illegible text
- Exceed 5MB file size
- Use pure white backgrounds

### App Card

**✅ DO:**
- Include both iOS and Android
- Use deep links for specific content
- Test install flow
- Provide clear app description

**❌ DON'T:**
- Use for web apps
- Include beta/unreleased apps
- Forget app ID
- Use HTTP URLs

### Player Card

**✅ DO:**
- Use HTTPS for player URL
- Provide poster image
- Test playback on mobile
- Implement proper CORS
- Include video dimensions

**❌ DON'T:**
- Require authentication
- Block iframe embedding
- Use non-standard players
- Forget mobile support

## Complete Examples

### Blog Post (Summary Large Image)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>10 Essential Hugo Performance Tips</title>

  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:site" content="@hugobest">
  <meta name="twitter:creator" content="@janedoe">
  <meta name="twitter:title" content="10 Essential Hugo Performance Tips">
  <meta name="twitter:description" content="Learn 10 proven techniques to make your Hugo site load 3x faster">
  <meta name="twitter:image" content="https://example.com/images/hugo-perf.jpg">
  <meta name="twitter:image:alt" content="Performance dashboard showing improvements">
</head>
</html>
```

### Author Profile (Summary)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Jane Doe - Author</title>

  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary">
  <meta name="twitter:site" content="@hugobest">
  <meta name="twitter:title" content="Jane Doe - Author">
  <meta name="twitter:description" content="Web developer and technical writer specializing in Hugo">
  <meta name="twitter:image" content="https://example.com/images/jane-square.jpg">
  <meta name="twitter:image:alt" content="Profile photo of Jane Doe">
</head>
</html>
```

### Video Tutorial (Player)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Hugo Basics Tutorial</title>

  <!-- Twitter Card -->
  <meta name="twitter:card" content="player">
  <meta name="twitter:site" content="@hugobest">
  <meta name="twitter:title" content="Introduction to Hugo Static Sites">
  <meta name="twitter:description" content="Learn Hugo basics in 10 minutes">
  <meta name="twitter:image" content="https://example.com/videos/poster.jpg">
  <meta name="twitter:player" content="https://example.com/embed/hugo-intro">
  <meta name="twitter:player:width" content="1280">
  <meta name="twitter:player:height" content="720">
  <meta name="twitter:player:stream" content="https://example.com/videos/hugo.mp4">
  <meta name="twitter:player:stream:content_type" content="video/mp4">
</head>
</html>
```

### Mobile App (App Card)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Hugo Tracker App</title>

  <!-- Twitter Card -->
  <meta name="twitter:card" content="app">
  <meta name="twitter:site" content="@hugotrac">
  <meta name="twitter:description" content="Track Hugo builds and deployments">

  <!-- iOS -->
  <meta name="twitter:app:name:iphone" content="Hugo Tracker">
  <meta name="twitter:app:id:iphone" content="123456789">
  <meta name="twitter:app:url:iphone" content="hugotrac://home">

  <!-- Android -->
  <meta name="twitter:app:name:googleplay" content="Hugo Tracker">
  <meta name="twitter:app:id:googleplay" content="com.example.hugotrac">
  <meta name="twitter:app:url:googleplay" content="hugotrac://home">
</head>
</html>
```

## Guidelines

### Essential

**All card types require:**
- twitter:card (specify type)
- twitter:title (max 70 characters)
- twitter:description (max 200 characters)

**Summary and summary_large_image require:**
- twitter:image (HTTPS URL)
- twitter:image:alt (accessibility)

**App card requires:**
- twitter:app:name:{platform}
- twitter:app:id:{platform}

**Player card requires:**
- twitter:player (embed URL)
- twitter:player:width
- twitter:player:height

### Recommended

**For better engagement:**
- twitter:site (@username)
- twitter:creator (@username for authors)
- Open Graph tags (fallback)
- Optimized images (< 500KB)
- Testing with Card Validator

### Advanced

**For maximum control:**
- Dynamic card type selection
- Section-specific defaults
- Conditional templates
- Fallback images by content type
- Deep linking (app cards)
- Custom player implementation

## Performance Checklist

**Before deploying:**

- [ ] Card type matches content format
- [ ] All required tags present
- [ ] Title under 70 characters
- [ ] Description under 200 characters
- [ ] Image meets size requirements (card-specific)
- [ ] Image file size optimized (< 500KB)
- [ ] HTTPS URLs used throughout
- [ ] Alt text provided
- [ ] Tested with Twitter Card Validator
- [ ] Tested on mobile device
- [ ] Open Graph tags included (fallback)
- [ ] @username attribution included

## Benefits

Increased Engagement. Rich previews drive 2-3x more clicks than plain links.

Content Flexibility. Different card types match different content formats.

Professional Appearance. Polished previews build brand credibility.

Mobile Optimized. Cards designed for mobile-first Twitter experience.

App Promotion. App cards drive mobile installs directly from tweets.

## Related

- [twitter-card-images.md](./twitter-card-images.md) - Image specifications and optimization
- [open-graph-meta-tags.md](./open-graph-meta-tags.md) - Open Graph Protocol (fallback tags)
- [social-media-image-generation.md](./social-media-image-generation.md) - Automated image generation
- [meta-descriptions-titles.md](./meta-descriptions-titles.md) - Title and description optimization
