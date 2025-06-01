# Theme Color Meta Tag

Meta tag that sets the browser UI color (address bar, status bar) on mobile devices and PWAs.

## Why It Matters

- Colors the mobile browser chrome to match your brand
- Used by Android Chrome, Samsung Internet, and other mobile browsers
- Works alongside the `theme_color` field in `site.webmanifest`

## HTML Meta Tag

```html
<meta name="theme-color" content="#2196F3">
```

## With Dark Mode Support

```html
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#1a1a1a" media="(prefers-color-scheme: dark)">
```

## Hugo Template

```go-html-template
<meta name="theme-color" content="{{ .Site.Params.themeColor | default "#ffffff" }}">
```

## Where Theme Color Appears

| Context | Source |
|---------|--------|
| Android Chrome address bar | `<meta name="theme-color">` |
| PWA title bar (standalone mode) | `theme_color` in manifest |
| Task switcher (Android) | `theme_color` in manifest |
| Safari 15+ (macOS/iOS) | `<meta name="theme-color">` |

## Relationship with Manifest

The `<meta>` tag overrides the manifest `theme_color` on a per-page basis. The manifest value is the default for the entire app.

```json
{ "theme_color": "#2196F3" }
```

## Pitfalls to Avoid

- Invalid color format (use hex `#RRGGBB`, not named colors)
- Forgetting to set both light and dark variants
- Mismatch between meta tag and manifest `theme_color`
- Very light color on light backgrounds making the status bar unreadable

## Related

- [webmanifest-pwa.md](./webmanifest-pwa.md)
- [favicon-modern-formats.md](./favicon-modern-formats.md)
