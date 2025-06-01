# browserconfig.xml

XML file configuring how your site appears when pinned to Windows Start Menu or Edge favorites.

## Why It Matters

- Provides branded tile appearance for Windows users
- Low effort: one XML file + one PNG image
- Less critical in 2026 but still supported by Windows 10/11 and Edge

## Minimal Setup

```xml
<?xml version="1.0" encoding="utf-8"?>
<browserconfig>
  <msapplication>
    <tile>
      <square150x150logo src="/mstile-150x150.png"/>
      <TileColor>#2196F3</TileColor>
    </tile>
  </msapplication>
</browserconfig>
```

```
static/
  browserconfig.xml
  mstile-150x150.png   (150x150, solid background matching TileColor)
```

## HTML Meta Tag (Alternative/Addition)

```html
<meta name="msapplication-TileColor" content="#2196F3">
<meta name="msapplication-config" content="/browserconfig.xml">
```

## All Tile Sizes (Optional)

| Size | Dimensions | Usage |
|------|-----------|-------|
| Small | 70x70 | Start Menu small |
| Medium | 150x150 | Start Menu default |
| Wide | 310x150 | Start Menu wide |
| Large | 310x310 | Start Menu large |

Only 150x150 (medium) is commonly seen. The rest are rarely used.

## Hugo Setup

Place `browserconfig.xml` in `static/` as a static file (simplest, rarely changes).

## Serving

- **Location:** `/browserconfig.xml` (root)
- **MIME type:** `application/xml; charset=utf-8`
- **Cache:** 1 day for XML, 1 year for tile images

## Pitfalls to Avoid

- Missing `<TileColor>` (Windows uses default color)
- Wrong MIME type (must be `application/xml`)
- Transparent tile image without matching TileColor
- Malformed XML (unclosed tags)
- File not at domain root

## Related

- [favicon-modern-formats.md](./favicon-modern-formats.md)
- [apple-touch-icons.md](./apple-touch-icons.md)
- [webmanifest-pwa.md](./webmanifest-pwa.md)
