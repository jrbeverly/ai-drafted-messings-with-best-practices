# Hugo Self-Hosted Dependencies

Self-host fonts, JS libraries, CSS frameworks, and icons for privacy, performance, and GDPR compliance.

## Why It Matters
- Eliminates external tracking requests (GDPR-friendly)
- Single-origin serving enables HTTP/2 multiplexing and removes DNS lookups
- No CDN downtime risk; works offline; full version control

## Self-Hosting Fonts
```bash
npm install @fontsource/inter
cp node_modules/@fontsource/inter/files/* static/fonts/inter/
```
```css
@font-face {
  font-family: 'Inter';
  font-weight: 400;
  font-display: swap;
  src: url('/fonts/inter/inter-v12-latin-regular.woff2') format('woff2');
}
```
- Prefer WOFF2 format (smallest size, universal support)
- Subset fonts with `pyftsubset` to reduce file size (Latin-only is often sufficient)
- Use variable fonts when you need multiple weights (single file)

## Self-Hosting JS Libraries
```bash
npm install alpinejs htmx.org
mkdir -p static/js/vendor
cp node_modules/alpinejs/dist/cdn.min.js static/js/vendor/alpine.js
```
Or process with Hugo Pipes for fingerprinting:
```go-html-template
{{ $alpine := resources.Get "js/vendor/alpine.js" | fingerprint }}
<script src="{{ $alpine.Permalink }}" integrity="{{ $alpine.Data.Integrity }}" defer></script>
```

## Self-Hosting CSS Frameworks
- **Tailwind CSS:** Install via npm, process with `postCSS` in Hugo Pipes
- **Bootstrap:** Import SCSS source via `@import` in your `main.scss`
- **Minimal approach:** Write a small custom framework (reset + variables + grid)

## Icons
Prefer SVG icon sprites over icon fonts:
```xml
<!-- assets/icons/sprite.svg -->
<svg xmlns="http://www.w3.org/2000/svg" style="display:none">
  <symbol id="icon-home" viewBox="0 0 24 24"><path d="..."/></symbol>
</svg>
```
```html
<svg class="icon"><use href="#icon-home"></use></svg>
```

## Package Management
```json
{
  "scripts": {
    "copy-assets": "cp -r node_modules/@fontsource/inter/files static/fonts/inter"
  }
}
```
Pin exact versions in `package.json`. Commit `package-lock.json`.

## Pitfalls
- Don't load entire libraries for one function (tree-shake or pick smaller alternatives)
- Don't load unused font weights (each weight adds ~20-50KB)
- Don't forget to update vendored dependencies periodically (`npm outdated`)
- Don't use icon fonts when SVG sprites work (better accessibility, smaller payload)
