# Lazy Loading Images

Defer loading of off-screen images until they approach the viewport using native `loading="lazy"`.

## Why It Matters

- Only above-fold images load immediately, reducing initial page weight
- Off-screen images never load if user does not scroll (saves bandwidth)
- Native `loading="lazy"` works without JavaScript (95%+ browser support)

## Core Pattern

```html
<!-- Above the fold (hero, LCP) -->
<img src="hero.jpg" loading="eager" fetchpriority="high" decoding="async" alt="Hero">

<!-- Below the fold -->
<img src="photo.jpg" loading="lazy" decoding="async" width="800" height="600" alt="Photo">
```

## Hugo Implementation

```go-html-template
{{/* First image eager, rest lazy */}}
{{ range $index, $page := .Pages }}
  {{ with .Resources.GetMatch "featured*" }}
    {{ $img := .Resize "800x jpg q85" }}
    <img src="{{ $img.RelPermalink }}"
      width="{{ $img.Width }}" height="{{ $img.Height }}"
      loading="{{ cond (eq $index 0) "eager" "lazy" }}"
      {{ if eq $index 0 }}fetchpriority="high"{{ end }}
      decoding="async" alt="{{ $page.Title }}">
  {{ end }}
{{ end }}
```

## Layout Shift Prevention

Always set `width` and `height` so the browser reserves space before load.

```go-html-template
{{ $resized := $image.Resize "800x" }}
<img src="{{ $resized.RelPermalink }}"
  width="{{ $resized.Width }}" height="{{ $resized.Height }}"
  loading="lazy" decoding="async" alt="Photo">
```

## LQIP (Low Quality Placeholder)

```go-html-template
{{ $lqip := $image.Resize "20x jpg q20" }}
{{ $full := $image.Resize "800x jpg q85" }}
<img src="{{ $lqip.RelPermalink }}" data-src="{{ $full.RelPermalink }}"
  class="lazy lqip" style="filter:blur(10px)" alt="Photo">
```

## Iframes and YouTube

```html
<iframe src="https://www.youtube.com/embed/..." loading="lazy" width="560" height="315"></iframe>
```

For better performance, use a YouTube facade pattern: show a thumbnail, load the iframe only on click.

## Key Recommendations

- `loading="lazy"` on all below-fold images and iframes
- `loading="eager"` + `fetchpriority="high"` on the LCP image
- `width` and `height` on every `<img>` to prevent CLS
- `decoding="async"` on all images
- LQIP or dominant color placeholder for perceived performance

## Pitfalls

- Lazy-loading above-the-fold images (hurts LCP)
- Forgetting `width`/`height` (causes layout shift)
- Using JavaScript-only lazy loading (native is sufficient)
- Lazy-loading tiny images under 5KB (not worth deferring)
- `loading="lazy"` on CSS background images (not supported)
