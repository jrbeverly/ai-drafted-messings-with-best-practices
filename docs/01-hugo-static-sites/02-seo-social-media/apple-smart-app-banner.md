# Apple Smart App Banner

Apple Smart App Banner. iOS app promotion. App Store link. Safari meta tag. App deep linking.

## Principle

Use Apple Smart App Banner to promote your iOS app from your website. Display a native banner in Safari on iOS devices. Link directly to specific content within your app using deep links.

## What is Apple Smart App Banner?

**Apple Smart App Banner:** Native banner displayed at the top of Safari on iOS when you have an associated app

**Purpose:**
- Promote your iOS app to website visitors
- Open app directly if already installed
- Link to App Store for installation
- Deep link to specific content in app

**Display behavior:**
- Shows banner at top of Safari
- User can dismiss (won't show again for that page)
- If app installed: "Open" button
- If not installed: "View" button (→ App Store)

## Meta Tag Syntax

### Basic Implementation

```html
<meta name="apple-itunes-app" content="app-id=123456789">
```

### With Affiliate Token

```html
<meta name="apple-itunes-app" content="app-id=123456789, affiliate-data=myAffiliateToken">
```

### With App Argument (Deep Link)

```html
<meta name="apple-itunes-app" content="app-id=123456789, app-argument=https://example.com/blog/post/">
```

### Full Implementation

```html
<meta name="apple-itunes-app" content="app-id=123456789, affiliate-data=myAffiliateToken, app-argument=https://example.com/blog/post/">
```

## Parameters

### app-id (Required)

**Your app's Apple App Store ID.**

```html
<meta name="apple-itunes-app" content="app-id=123456789">
```

**How to find your App ID:**
1. Go to App Store Connect
2. Select your app
3. Find Apple ID in App Information
4. Or look at your App Store URL: `apps.apple.com/app/id123456789`

### affiliate-data (Optional)

**iTunes affiliate token for commission tracking.**

```html
<meta name="apple-itunes-app" content="app-id=123456789, affiliate-data=myAffiliateToken">
```

**Use if:** You're part of Apple's affiliate program

### app-argument (Optional)

**Deep link URL or argument passed to app when opened.**

```html
<meta name="apple-itunes-app" content="app-id=123456789, app-argument=https://example.com/blog/post/">
```

**Use for:**
- Opening specific content in app
- Matching website page to app screen
- Continuing user's journey in app

**The app receives this argument** and can navigate to relevant content.

## Hugo Implementation

### Basic Template

**layouts/partials/head/apple-app-banner.html:**

```go-html-template
{{/* Apple Smart App Banner */}}
{{ with .Site.Params.apple_app_id }}
  {{ $content := printf "app-id=%s" . }}

  {{ with $.Site.Params.apple_affiliate_data }}
    {{ $content = printf "%s, affiliate-data=%s" $content . }}
  {{ end }}

  {{/* Deep link to current page */}}
  {{ if $.Site.Params.apple_app_deep_link }}
    {{ $content = printf "%s, app-argument=%s" $content $.Permalink }}
  {{ end }}

  <meta name="apple-itunes-app" content="{{ $content }}">
{{ end }}
```

### Site Configuration

**config.toml:**

```toml
[params]
  # Apple Smart App Banner
  apple_app_id = "123456789"
  apple_affiliate_data = "myAffiliateToken"  # Optional
  apple_app_deep_link = true  # Pass current URL as app-argument
```

### Conditional Display

**Only show on specific pages:**

```go-html-template
{{ if and .Site.Params.apple_app_id (not .Params.hide_app_banner) }}
  {{ $content := printf "app-id=%s" .Site.Params.apple_app_id }}

  {{/* Only add deep link for content pages */}}
  {{ if .IsPage }}
    {{ $content = printf "%s, app-argument=%s" $content .Permalink }}
  {{ end }}

  <meta name="apple-itunes-app" content="{{ $content }}">
{{ end }}
```

**Hide on specific pages:**

```yaml
---
title: "About Us"
hide_app_banner: true
---
```

### Page-Specific App Arguments

```go-html-template
{{ with .Site.Params.apple_app_id }}
  {{ $content := printf "app-id=%s" . }}

  {{/* Custom app argument from front matter */}}
  {{ with $.Params.app_argument }}
    {{ $content = printf "%s, app-argument=%s" $content . }}
  {{ else if $.IsPage }}
    {{/* Default: pass page URL */}}
    {{ $content = printf "%s, app-argument=%s" $content $.Permalink }}
  {{ end }}

  <meta name="apple-itunes-app" content="{{ $content }}">
{{ end }}
```

**Front matter:**

```yaml
---
title: "Hugo Performance Tips"
app_argument: "myapp://blog/hugo-tips"
---
```

## Include in Layout

**layouts/_default/baseof.html:**

```go-html-template
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{{ .Title }}</title>

  {{/* Apple Smart App Banner */}}
  {{ partial "head/apple-app-banner.html" . }}

  {{/* Other head elements */}}
</head>
```

## Deep Linking

### URL Scheme

**Pass custom URL scheme:**

```html
<meta name="apple-itunes-app" content="app-id=123456789, app-argument=myapp://blog/hugo-tips">
```

**App handles:** `myapp://blog/hugo-tips`

### Universal Links

**Pass web URL (if app supports Universal Links):**

```html
<meta name="apple-itunes-app" content="app-id=123456789, app-argument=https://example.com/blog/hugo-tips/">
```

**App handles:** `https://example.com/blog/hugo-tips/`

### Hugo Auto Deep Link

```go-html-template
{{/* Pass current page URL as deep link */}}
<meta name="apple-itunes-app" content="app-id={{ .Site.Params.apple_app_id }}, app-argument={{ .Permalink }}">
```

## Testing

### Safari on iOS

**Test on real device or simulator:**
1. Deploy page with meta tag
2. Open in Safari on iOS
3. Banner appears at top of page
4. Click "View" (if app not installed) → goes to App Store
5. Click "Open" (if app installed) → opens app

### Simulator

**Xcode iOS Simulator:**
1. Open Simulator
2. Open Safari
3. Navigate to your page
4. Banner should appear

### Validation

```bash
# Check meta tag exists
curl -s https://example.com/ | grep 'apple-itunes-app'

# Expected output:
# <meta name="apple-itunes-app" content="app-id=123456789, app-argument=https://example.com/">
```

## Best Practices

**✅ DO:**
- Use correct App Store ID
- Include deep links (app-argument) for relevant pages
- Test on real iOS device
- Use Universal Links for modern apps
- Respect user dismissal (don't force banner)
- Include only on pages relevant to app

**❌ DON'T:**
- Include on every page (annoying)
- Use wrong App Store ID
- Forget to test deep links
- Include multiple apple-itunes-app meta tags
- Use for apps not in the App Store

## Guidelines

### Essential

**Minimum implementation:**
- `app-id` parameter (required)
- Valid App Store ID
- Meta tag in `<head>`

### Recommended

**For better conversion:**
- `app-argument` for deep linking
- Conditional display (content pages only)
- Page-specific deep links
- Testing on real iOS devices
- Hide on pages where app isn't relevant

### Advanced

**For maximum app installs:**
- Universal Links integration
- Affiliate tracking
- Custom URL schemes
- Analytics on banner interactions
- A/B testing banner placement

## Benefits

Native Experience. Apple-designed banner fits iOS Safari naturally.

Smart Behavior. Detects if app is installed or not.

Deep Linking. Users go directly to relevant content in app.

Non-Intrusive. Users can dismiss, won't reappear for that page.

Zero JavaScript. Pure meta tag, no performance impact.

## Related

- [meta-tags-comprehensive.md](./meta-tags-comprehensive.md) - All HTML meta tags
- [twitter-card-types.md](./twitter-card-types.md) - Twitter App Card (similar concept)
- [rel-attributes.md](./rel-attributes.md) - Link relationship attributes
