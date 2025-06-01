# Web App Manifest (PWA)

JSON file (`site.webmanifest`) describing how your web app behaves when installed on a user's device.

## Why It Matters

- Enables "Add to Home Screen" / install prompts on Android and desktop
- Defines app name, icons, display mode, and theme colors
- Required (along with a service worker and HTTPS) for full PWA installability

## Minimal Manifest

```json
{
  "name": "Example App",
  "short_name": "Example",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#2196F3",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

## HTML Link Tag

```html
<link rel="manifest" href="/site.webmanifest">
```

## Key Fields

| Field | Purpose |
|-------|---------|
| `name` | Full app name (install dialog) |
| `short_name` | Label under icon (max ~12 chars) |
| `start_url` | URL opened on launch (`/?source=pwa` for analytics) |
| `display` | `standalone` (app-like), `fullscreen`, `minimal-ui`, `browser` |
| `background_color` | Splash screen background |
| `theme_color` | Browser UI / address bar color |
| `icons` | Minimum: 192x192 and 512x512 PNG |

## Maskable Icons

For adaptive icon shapes (circle, squircle) on Android, provide a maskable icon with content in the center 80%:

```json
{ "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any maskable" }
```

## Optional Fields

- `description` -- app description
- `scope` -- navigation scope (URLs outside open in browser)
- `orientation` -- `portrait-primary`, `landscape`, `any`
- `shortcuts` -- app shortcuts menu entries
- `screenshots` -- for richer install UI
- `categories` -- `["productivity", "business"]`

## Hugo Setup

Place `site.webmanifest` in `static/`. Create `static/icons/` with 192 and 512 PNGs.

## Serving Requirements

- **MIME type:** `application/manifest+json`
- **Protocol:** HTTPS required
- **Cache:** 1-24 hours

## PWA Installability Checklist

- Valid manifest linked in HTML
- `name`, `icons` (192 + 512), `start_url`, `display` defined
- Service worker registered
- Served over HTTPS

## Pitfalls to Avoid

- Missing 192x192 or 512x512 icons (Chrome requires both)
- Wrong MIME type (`text/plain` instead of `application/manifest+json`)
- Relative `start_url` like `index.html` (use `/`)
- Not serving over HTTPS (PWA requires it)
- `short_name` too long (gets truncated on home screen)

## Related

- [favicon-modern-formats.md](./favicon-modern-formats.md)
- [apple-touch-icons.md](./apple-touch-icons.md)
