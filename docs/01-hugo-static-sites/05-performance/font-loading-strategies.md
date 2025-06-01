# Font Loading Strategies

Self-host fonts in WOFF2 with `font-display: swap`, preload critical weights, and match fallback metrics to prevent layout shift.

## Why It Matters

- Without a strategy, browsers hide text up to 3 seconds while fonts load (FOIT)
- Self-hosting eliminates 2 DNS lookups + 2 TLS handshakes vs Google Fonts
- Subsetting reduces font payload from ~100KB to ~15-20KB per weight

## Core Setup

```css
@font-face {
  font-family: 'Inter';
  font-style: normal;
  font-weight: 400;
  font-display: swap;
  src: url('/fonts/inter-regular.woff2') format('woff2');
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+2000-206F, U+20AC;
}
```

## font-display Values

| Value | Behavior | Use Case |
|---|---|---|
| `swap` | Show fallback immediately, swap when ready | Body text, headings (recommended) |
| `optional` | Use font only if cached | Zero layout shift priority |
| `fallback` | Brief ~100ms hidden, then fallback | Balance FOIT/FOUT |
| `block` | Hide text up to 3s | Icon fonts only |

## Preload Critical Fonts (1-2 max)

```html
<link rel="preload" href="/fonts/inter-regular.woff2"
  as="font" type="font/woff2" crossorigin>
```

`crossorigin` is **required** even for same-origin fonts -- omitting it causes a double download.

## Fallback Font Metrics Matching

```css
@font-face {
  font-family: 'Inter Fallback';
  src: local('Arial');
  size-adjust: 107.64%;
  ascent-override: 90.49%;
  descent-override: 22.56%;
  line-gap-override: 0%;
}
body { font-family: 'Inter', 'Inter Fallback', system-ui, sans-serif; }
```

Generate values with `npx fontaine ./static/fonts/inter-regular.woff2`.

## Subsetting

```bash
pyftsubset Inter-Regular.ttf \
  --output-file=inter-regular-latin.woff2 \
  --flavor=woff2 \
  --unicodes='U+0000-00FF,U+0131,U+0152-0153,U+2000-206F,U+20AC'
```

## Variable Fonts

Use when 3+ weights are needed. One file (~40-60KB) replaces 4 separate files (~60-80KB).

```css
@font-face {
  font-family: 'Inter';
  font-weight: 100 900;
  font-display: swap;
  src: url('/fonts/inter-variable.woff2') format('woff2-variations');
}
```

## System Font Stack Alternative

Zero font files, zero loading delay, zero layout shift:

```css
body { font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; }
```

## Key Recommendations

- Self-host WOFF2 only (97%+ support, no WOFF/TTF fallback needed)
- `font-display: swap` on all text `@font-face` declarations
- Preload only 1-2 critical font files
- Subset to Latin-only for English sites
- Define fallback font with `size-adjust`/`ascent-override` to minimize CLS
- Use Hugo Pipes `fingerprint` for immutable caching
- Inline `@font-face` declarations in `<style>` to avoid render-blocking CSS

## Pitfalls

- Loading fonts from Google Fonts CDN (extra DNS, TLS, privacy)
- Forgetting `crossorigin` on font preloads (causes double download)
- Preloading more than 2-3 font files (saturates bandwidth)
- Using `font-display: auto` or `block` for text (causes FOIT)
- Skipping fallback font metrics matching (causes visible layout shift)
