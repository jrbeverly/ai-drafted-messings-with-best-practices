# Favicon Modern Formats

Modern favicon setup using SVG, PNG, and ICO for cross-browser and cross-device compatibility.

## Why It Matters

- Browsers, bookmarks, and OS shells all request favicons
- SVG enables dark mode support and infinite scalability
- Multiple formats ensure legacy and modern browser coverage

## Recommended Minimal Setup

```html
<link rel="icon" href="/favicon.svg" type="image/svg+xml">
<link rel="icon" href="/favicon.png" type="image/png">
<link rel="icon" href="/favicon.ico" sizes="any">
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
<link rel="manifest" href="/site.webmanifest">
```

## File Inventory

```
static/
  favicon.svg           # Modern browsers (scalable, dark mode)
  favicon.png           # Fallback (32x32)
  favicon.ico           # Legacy (multi-size: 16, 32, 48)
  apple-touch-icon.png  # iOS (180x180)
  icons/
    icon-192.png        # PWA (192x192)
    icon-512.png        # PWA splash (512x512)
```

## SVG with Dark Mode

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 100 100">
  <style>
    circle { fill: #2196F3; }
    @media (prefers-color-scheme: dark) {
      circle { fill: #64B5F6; }
    }
  </style>
  <circle cx="50" cy="50" r="40"/>
</svg>
```

## Generate from Source Image

```bash
convert logo.png -resize 32x32 favicon.png
convert logo.png -resize 16x16 f16.png
convert logo.png -resize 32x32 f32.png
convert logo.png -resize 48x48 f48.png
convert f16.png f32.png f48.png favicon.ico
```

## Browser Support

| Format | Chrome | Firefox | Safari | Edge | IE |
|--------|--------|---------|--------|------|----|
| SVG | 94+ | 41+ | 9+ | 79+ | No |
| PNG | All | All | All | All | 11+ |
| ICO | All | All | All | All | All |

## Caching

Set long cache headers (1 year, immutable) since favicons rarely change:

```
Cache-Control: public, max-age=31536000, immutable
```

## Target File Sizes

- `favicon.svg`: < 1 KB
- `favicon.png` (32x32): < 1 KB
- `favicon.ico`: < 5 KB
- `apple-touch-icon.png`: < 10 KB

## Pitfalls to Avoid

- Only providing `favicon.ico` (missing modern SVG/PNG)
- Wrong MIME types (`image/svg+xml` for SVG, `image/png` for PNG, `image/x-icon` for ICO)
- Oversized favicon PNGs (100 KB for a 32x32 is excessive)
- Complex designs that are unrecognizable at 16x16

## Related

- [apple-touch-icons.md](./apple-touch-icons.md)
- [webmanifest-pwa.md](./webmanifest-pwa.md)
- [browserconfig-xml.md](./browserconfig-xml.md)
