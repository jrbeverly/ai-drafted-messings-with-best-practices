# Image Optimization & Formats

Serve images in modern formats (AVIF, WebP) with responsive sizing via Hugo's build-time image processing.

## Why It Matters

- AVIF/WebP reduce image size 50-80% vs unoptimized JPEG/PNG
- Proper `width`/`height` attributes prevent layout shift (CLS)
- `srcset` with multiple sizes avoids sending oversized images to mobile

## Format Selection

| Format | Compression vs JPEG | Use Case |
|---|---|---|
| AVIF | 50%+ smaller | Photos (primary) |
| WebP | 25-35% smaller | Photos (fallback) |
| JPEG | Baseline | Universal fallback |
| SVG | Vector (tiny) | Icons, logos, illustrations |

## Hugo Image Processing

```go-html-template
{{ $image := resources.Get "images/hero.jpg" }}
{{ $webp := $image.Resize "800x webp q80" }}
{{ $avif := $image.Resize "800x avif q65" }}
{{ $small := $image.Resize "400x" }}
{{ $large := $image.Resize "1200x" }}
{{ $thumbnail := $image.Fill "300x300 center" }}
```

## Responsive Image Pattern (`<picture>`)

```go-html-template
{{ $widths := slice 400 600 800 1200 }}
<picture>
  <source type="image/avif"
    srcset="{{ range $widths }}...{{ $image.Resize (printf "%dx avif q70" .) }}...{{ end }}"
    sizes="(min-width: 768px) 720px, 100vw">
  <source type="image/webp" srcset="..." sizes="...">
  <img src="{{ $fallback.RelPermalink }}"
    width="{{ $fallback.Width }}" height="{{ $fallback.Height }}"
    loading="lazy" decoding="async" alt="...">
</picture>
```

## Quality Settings

| Format | Quality | Notes |
|---|---|---|
| AVIF | q60-70 | Smallest file, excellent visual quality |
| WebP | q75-85 | Good balance |
| JPEG | q80-90 | Fallback only |

```toml
# hugo.toml
[imaging]
  quality = 85
  resampleFilter = "Lanczos"
  [imaging.exif]
    disableLatLong = true  # Strip GPS data
```

## Render Hook for Markdown Images

Place in `layouts/_default/_markup/render-image.html` to auto-optimize all `![alt](image.jpg)` in content. Generates WebP srcset with JPEG fallback, `loading="lazy"`, and `decoding="async"` automatically.

## Key Recommendations

- Use `<picture>` with AVIF > WebP > JPEG source order
- Set explicit `width` and `height` on all `<img>` tags
- Use `loading="eager"` + `fetchpriority="high"` for LCP image only
- Use `loading="lazy"` + `decoding="async"` for everything else
- Generate sizes at 400, 600, 800, 1000, 1200 widths
- Use page bundles for content images

## Pitfalls

- Serving full-resolution images to mobile (use `srcset`)
- Using PNG for photographs (use WebP/AVIF)
- Skipping `width`/`height` (causes layout shift)
- Using `loading="lazy"` on above-the-fold images (hurts LCP)
- Using CSS to resize images (wastes bandwidth)
