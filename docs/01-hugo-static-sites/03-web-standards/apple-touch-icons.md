# Apple Touch Icons

PNG icons displayed when users add your website to their iOS/iPadOS home screen.

## Why It Matters

- Without it, iOS takes an ugly screenshot as the bookmark icon
- One 180x180 PNG covers all modern Apple devices (iOS scales down automatically)
- iOS auto-detects `/apple-touch-icon.png` at root even without a link tag

## Minimal Setup (Recommended)

```html
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
```

```
static/
  apple-touch-icon.png  (180x180, PNG, solid background)
```

That is all you need. Providing many sizes (167, 152, 120) is unnecessary for most sites.

## Design Requirements

- **Size:** 180x180 pixels
- **Format:** PNG
- **Background:** Solid color (no transparency -- older iOS shows black behind transparent areas)
- **Safe area:** ~160x160 center zone (iOS adds 18% corner radius)
- **No rounded corners:** iOS adds them automatically

## Generate from Source

```bash
convert logo.png -resize 180x180 \
  -background "#FFFFFF" -gravity center -extent 180x180 \
  apple-touch-icon.png
```

## Optional iOS Web App Meta Tags

```html
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="My App">
```

## Caching

```
Cache-Control: public, max-age=31536000, immutable
Content-Type: image/png
```

## Pitfalls to Avoid

- Transparent background (shows black on older iOS)
- Wrong dimensions (192x192 is for Android PWA, not Apple)
- Manually adding rounded corners (iOS does this)
- Providing 10 different sizes (180x180 alone is sufficient)
- Using JPEG format (must be PNG)

## Related

- [favicon-modern-formats.md](./favicon-modern-formats.md)
- [webmanifest-pwa.md](./webmanifest-pwa.md)
