# Critical CSS & Render-Blocking Elimination

Inline minimal above-the-fold CSS in `<head>` and load the full stylesheet asynchronously to eliminate render-blocking.

## Why It Matters

- Browsers block painting until all `<link>` CSS downloads and parses
- Inlining critical CSS enables first paint without waiting for external CSS
- Combined with PurgeCSS, reduces CSS payload by 80-95%

## Inline Critical CSS in Hugo

```go-html-template
{{ $critical := resources.Get "css/critical.css" | minify }}
<style>{{ $critical.Content | safeCSS }}</style>
```

## Async Load Full Stylesheet

```go-html-template
{{ $full := resources.Get "css/main.css" | postCSS | minify | fingerprint }}
<link rel="preload" href="{{ $full.RelPermalink }}" as="style"
  onload="this.onload=null;this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="{{ $full.RelPermalink }}"></noscript>
```

Alternative: `media="print" onload="this.media='all'"` (universal browser support).

## PurgeCSS with Hugo

```toml
# hugo.toml
[build.buildStats]
  enable = true  # Generates hugo_stats.json for PurgeCSS
```

```js
// postcss.config.js (production only)
const purgecss = require('@fullhuman/postcss-purgecss')({
  content: ['./hugo_stats.json'],
  defaultExtractor: (content) => {
    const els = JSON.parse(content).htmlElements;
    return [...(els.tags || []), ...(els.classes || []), ...(els.ids || [])];
  },
  safelist: { standard: [/^is-/, /^has-/, 'active'] },
});
```

## Page-Specific Critical CSS

```go-html-template
{{ $criticalFile := "css/critical.css" }}
{{ if .IsHome }}{{ $criticalFile = "css/critical-home.css" }}{{ end }}
{{ if eq .Type "blog" }}{{ $criticalFile = "css/critical-blog.css" }}{{ end }}
```

## CSP Considerations

Inline `<style>` requires either `'unsafe-inline'`, a nonce, or a hash in your Content-Security-Policy. For static sites, `'unsafe-inline'` is simplest; use nonce/hash for stronger security.

## Key Recommendations

- Inline critical CSS in `<head>` (keep under 14KB for first TCP round trip)
- Async load full CSS with `preload`/`onload` pattern
- Always include `<noscript>` fallback
- Enable `build.buildStats` for PurgeCSS integration
- Use `fingerprint` for long-term caching of async stylesheets
- Separate critical CSS files for distinct page layouts

## Pitfalls

- Inlining the entire stylesheet (defeats the purpose)
- Forgetting the `<noscript>` fallback
- Using `@import` in critical CSS (creates blocking requests)
- Skipping PurgeCSS in production
- Loading web fonts in critical CSS (blocks rendering separately)
