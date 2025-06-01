# Hugo Asset Pipeline (Hugo Pipes)

Built-in SCSS compilation, JS bundling (ESBuild), image processing, minification, and fingerprinting.

## Why It Matters
- Hugo Pipes eliminates external build tools for most asset processing needs
- Built-in image resizing and WebP conversion optimize Core Web Vitals
- Fingerprinting + SRI provides cache busting and tamper protection

## Pipeline Pattern
All assets live in `assets/` and are processed in templates:
```go-html-template
{{ $css := resources.Get "scss/main.scss" | toCSS | postCSS | minify | fingerprint }}
<link rel="stylesheet" href="{{ $css.Permalink }}" integrity="{{ $css.Data.Integrity }}">
```

## SCSS/CSS
```go-html-template
{{ $opts := dict "outputStyle" "compressed" "enableSourceMap" (not hugo.IsProduction) }}
{{ $style := resources.Get "scss/main.scss" | toCSS $opts }}
```
For PostCSS/Tailwind: `npm install -D postcss postcss-cli autoprefixer tailwindcss`

## JavaScript (ESBuild)
```go-html-template
{{ $js := resources.Get "js/main.js" | js.Build (dict "minify" true "target" "es2018") | fingerprint }}
<script src="{{ $js.Permalink }}" integrity="{{ $js.Data.Integrity }}" defer></script>
```
- Supports ES6 imports natively
- Inject Hugo params: `import * as params from '@params'` with `"params"` dict option

## Image Processing
```go-html-template
{{ $img := resources.Get "images/hero.jpg" }}
{{ $webp := $img.Resize "800x webp" }}
{{ $jpeg := $img.Resize "800x" }}
<picture>
  <source srcset="{{ $webp.RelPermalink }}" type="image/webp">
  <img src="{{ $jpeg.RelPermalink }}" alt="Hero" loading="lazy">
</picture>
```
Operations: `.Resize "800x"`, `.Fit "800x600"`, `.Fill "800x600 center"`
Quality: append `q85`, format: append `webp` or `jpg`

## Environment-Specific Processing
```go-html-template
{{ if hugo.IsProduction }}
  {{ $css = $css | minify | fingerprint }}
{{ end }}
```

## Pitfalls
- Don't process assets inside `{{ range }}` loops -- assign to variable first
- Don't skip `defer` or `async` on script tags
- Don't serve unoptimized original images -- always resize
- Don't inline large stylesheets -- only inline critical CSS
- Don't forget `postCSS` step if using autoprefixer or Tailwind
