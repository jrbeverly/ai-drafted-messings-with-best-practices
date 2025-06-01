# Asset Bundling & Minification

Process all CSS and JS through Hugo Pipes to bundle, minify, fingerprint, and generate SRI hashes at build time.

## Why It Matters

- Reduces CSS by 80-95% and JS by 40-70% through minification, tree shaking, and PurgeCSS
- Content-hashed filenames enable immutable caching (1 year) with automatic cache busting
- SRI hashes verify assets have not been tampered with in transit

## Pipeline Order

```
CSS: Sass/SCSS compile -> PostCSS (Tailwind, Autoprefixer) -> PurgeCSS -> Minify -> Fingerprint
JS:  js.Build (ESBuild: bundle, tree shake, transpile) -> Minify -> Fingerprint
```

Always: compile first, purge/minify in production only, fingerprint last.

## CSS Pipeline

```go-html-template
{{ $style := resources.Get "css/main.css" | postCSS }}
{{ if hugo.IsProduction }}
  {{ $style = $style | minify | fingerprint }}
{{ end }}
<link rel="stylesheet" href="{{ $style.RelPermalink }}"
  {{ with $style.Data.Integrity }}integrity="{{ . }}" crossorigin="anonymous"{{ end }}>
```

## JS Pipeline (ESBuild)

```go-html-template
{{ $js := resources.Get "js/main.js" | js.Build (dict
  "target" "es2020" "minify" hugo.IsProduction
  "sourceMap" (cond hugo.IsProduction "" "inline")
) }}
{{ if hugo.IsProduction }}{{ $js = $js | fingerprint }}{{ end }}
<script src="{{ $js.RelPermalink }}" defer
  {{ with $js.Data.Integrity }}integrity="{{ . }}" crossorigin="anonymous"{{ end }}></script>
```

ESBuild handles bundling, tree shaking, TypeScript, and minification natively in Hugo.

## Concatenating Files

```go-html-template
{{ $bundle := slice $reset $base $components | resources.Concat "css/bundle.css" }}
{{ $bundle = $bundle | minify | fingerprint }}
```

Apply PostCSS/minify/fingerprint after concatenation, not before.

## Fingerprinting and SRI

```go-html-template
{{ $style := resources.Get "css/main.css" | minify | fingerprint }}
{{/* Output: /css/main.min.a1b2c3d4e5.css */}}
<link rel="stylesheet" href="{{ $style.RelPermalink }}"
  integrity="{{ $style.Data.Integrity }}" crossorigin="anonymous">
```

Cache headers for fingerprinted assets: `Cache-Control: public, max-age=31536000, immutable`

## Inline Small Assets

```go-html-template
{{ if lt (len $style.Content) 1024 }}
  <style>{{ $style.Content | safeCSS }}</style>  {{/* < 1KB: inline */}}
{{ else }}
  <link rel="stylesheet" href="{{ $style.RelPermalink }}">  {{/* external */}}
{{ end }}
```

## Key Recommendations

- `minify | fingerprint` (in that order) for every production asset
- Use `js.Build` (ESBuild) for all JS bundling instead of Webpack/Rollup
- `defer` on all script tags to prevent render blocking
- `hugo --minify` for HTML output minification
- Source maps in development only
- SRI hashes (`integrity` + `crossorigin="anonymous"`) on all external CSS/JS

## Pitfalls

- Fingerprinting before minifying (hash does not reflect minified content)
- Using external bundlers when Hugo Pipes covers the use case
- Inlining files over 1KB (should be external cached resources)
- Enabling source maps in production (exposes source, increases payload)
- Forgetting `crossorigin` when using `integrity` (causes SRI failure)
- Applying PostCSS before Sass compilation
