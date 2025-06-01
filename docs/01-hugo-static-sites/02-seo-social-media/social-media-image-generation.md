# Social Media Image Generation

Automated OG image generation. Dynamic social cards. Text-on-image overlays. Hugo image processing. External image services.

## Principle

Automate social media image generation for consistent branding across all shared content. Use dynamic text overlays to create unique images per page. Reduce manual design effort while maintaining visual quality.

## Why Automate Image Generation?

### The Problem

**Manual image creation:**
- Time-consuming (15-30 min per image)
- Inconsistent branding
- Often skipped for low-priority content
- Doesn't scale with content volume

**Automated generation:**
- ✅ Every page has a unique social image
- ✅ Consistent brand identity
- ✅ Zero manual effort per page
- ✅ Scales to thousands of pages

### Image Requirements

**Open Graph:** 1200×630 pixels (1.91:1)
**Twitter Card:** 1200×628 pixels (1.91:1)
**LinkedIn:** 1200×627 pixels (1.91:1)
**Pinterest:** 1000×1500 pixels (2:3)

**Universal size:** 1200×630 works across all major platforms.

## Approaches

### 1. Hugo Image Processing (Built-in)

**Resize and crop existing images:**

```go-html-template
{{ $image := resources.Get "images/hero.jpg" }}
{{ $og := $image.Fill "1200x630 center jpg q85" }}
<meta property="og:image" content="{{ $og.Permalink }}">
```

**Pros:** No external dependencies, fast, built-in
**Cons:** Can't add text overlays, only crops/resizes

### 2. External Image Generation APIs

**Services that generate images with text overlays:**

- Cloudinary (URL-based transformations)
- imgix (URL-based transformations)
- Bannerbear (template-based)
- Placid (template-based)

**Example with Cloudinary:**

```go-html-template
{{ $title := .Title | urlquery }}
{{ $ogURL := printf "https://res.cloudinary.com/yourcloud/image/upload/w_1200,h_630,c_fill/l_text:Arial_72_bold:%s,co_rgb:FFFFFF,g_center/social-template.jpg" $title }}
<meta property="og:image" content="{{ $ogURL }}">
```

### 3. Build-Time Generation (Node.js/Puppeteer)

**Generate images during build using headless browser:**

```javascript
// scripts/generate-og-images.js
const puppeteer = require('puppeteer');

async function generateOGImage(title, outputPath) {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.setViewport({ width: 1200, height: 630 });

  await page.setContent(`
    <html>
    <body style="margin:0; width:1200px; height:630px; display:flex; align-items:center; justify-content:center; background: linear-gradient(135deg, #1a1a2e, #16213e); font-family: system-ui;">
      <h1 style="color:white; font-size:72px; text-align:center; padding:60px; max-width:1000px;">${title}</h1>
    </body>
    </html>
  `);

  await page.screenshot({ path: outputPath, type: 'jpeg', quality: 85 });
  await browser.close();
}
```

**Pros:** Full design control, custom fonts, complex layouts
**Cons:** Slower build, requires Node.js, heavier dependencies

### 4. SVG-Based Generation

**Generate SVG templates, convert to PNG:**

```svg
<svg width="1200" height="630" xmlns="http://www.w3.org/2000/svg">
  <rect width="1200" height="630" fill="#1a1a2e"/>
  <text x="600" y="280" text-anchor="middle" fill="white"
        font-family="system-ui" font-size="72" font-weight="bold">
    {{TITLE}}
  </text>
  <text x="600" y="380" text-anchor="middle" fill="#888"
        font-family="system-ui" font-size="36">
    {{SITE_NAME}}
  </text>
</svg>
```

**Convert with ImageMagick:**

```bash
convert template.svg -resize 1200x630 og-image.png
```

### 5. Canvas-Based (Go/Hugo Module)

**Use Go image library via Hugo module:**

Not natively supported in Hugo templates, but possible via custom Hugo module or external Go program.

## Hugo Image Processing

### Basic Crop and Resize

**layouts/partials/head/og-image.html:**

```go-html-template
{{ $ogImage := "" }}

{{/* 1. Check for explicit OG image */}}
{{ with .Params.og_image }}
  {{ $resource := resources.Get . }}
  {{ if $resource }}
    {{ $og := $resource.Fill "1200x630 center jpg q85" }}
    {{ $ogImage = $og.Permalink }}
  {{ else }}
    {{ $ogImage = . | absURL }}
  {{ end }}

{{/* 2. Check for page images */}}
{{ else }}
  {{ with .Params.images }}
    {{ $img := index . 0 }}
    {{ $resource := resources.Get $img }}
    {{ if $resource }}
      {{ $og := $resource.Fill "1200x630 center jpg q85" }}
      {{ $ogImage = $og.Permalink }}
    {{ else }}
      {{ $ogImage = $img | absURL }}
    {{ end }}
  {{ end }}
{{ end }}

{{/* 3. Check page bundle */}}
{{ if not $ogImage }}
  {{ $bundleImage := .Resources.GetMatch "{og,featured,hero,cover}*" }}
  {{ with $bundleImage }}
    {{ $og := .Fill "1200x630 center jpg q85" }}
    {{ $ogImage = $og.Permalink }}
  {{ end }}
{{ end }}

{{/* 4. Section default */}}
{{ if not $ogImage }}
  {{ $sectionDefault := printf "images/og/%s-default.jpg" .Section }}
  {{ $resource := resources.Get $sectionDefault }}
  {{ with $resource }}
    {{ $og := .Fill "1200x630 center jpg q85" }}
    {{ $ogImage = $og.Permalink }}
  {{ end }}
{{ end }}

{{/* 5. Site default */}}
{{ if not $ogImage }}
  {{ with .Site.Params.og_image }}
    {{ $ogImage = . | absURL }}
  {{ end }}
{{ end }}

{{/* Output */}}
{{ with $ogImage }}
  <meta property="og:image" content="{{ . }}">
  <meta property="og:image:width" content="1200">
  <meta property="og:image:height" content="630">
  <meta name="twitter:image" content="{{ . }}">
{{ end }}
```

### Image Processing Options

**Hugo image processing methods:**

```go-html-template
{{/* Fill: Resize and crop to exact dimensions */}}
{{ $image.Fill "1200x630 center jpg q85" }}

{{/* Fit: Resize to fit within dimensions (may not fill) */}}
{{ $image.Fit "1200x630 jpg q85" }}

{{/* Resize: Resize to width or height */}}
{{ $image.Resize "1200x jpg q85" }}

{{/* Crop: Crop to exact dimensions from anchor point */}}
{{ $image.Crop "1200x630 center" }}
```

**Anchor points for Fill/Crop:**

```
TopLeft    Top    TopRight
Left       Center Right
BottomLeft Bottom BottomRight
Smart (content-aware)
```

**Quality settings:**

```go-html-template
{{ $image.Fill "1200x630 center jpg q85" }}  <!-- JPEG quality 85% -->
{{ $image.Fill "1200x630 center webp q80" }} <!-- WebP quality 80% -->
{{ $image.Fill "1200x630 center png" }}      <!-- PNG (lossless) -->
```

### Image Filters

**Apply filters for consistent look:**

```go-html-template
{{ $image := resources.Get "images/hero.jpg" }}

{{/* Darken image for text overlay readability */}}
{{ $darkened := $image.Filter (images.Brightness -30) }}
{{ $og := $darkened.Fill "1200x630 center jpg q85" }}

{{/* Blur background */}}
{{ $blurred := $image.Filter (images.GaussianBlur 5) }}
{{ $og := $blurred.Fill "1200x630 center jpg q85" }}

{{/* Grayscale */}}
{{ $gray := $image.Filter (images.Grayscale) }}
{{ $og := $gray.Fill "1200x630 center jpg q85" }}

{{/* Multiple filters */}}
{{ $processed := $image.Filter
  (images.GaussianBlur 3)
  (images.Brightness -20)
  (images.Contrast 10)
}}
{{ $og := $processed.Fill "1200x630 center jpg q85" }}
```

## External Service Integration

### Cloudinary

**URL-based image transformation:**

**config.toml:**

```toml
[params]
  cloudinary_cloud = "yourcloud"
  cloudinary_template = "social-template"
```

**Template:**

```go-html-template
{{ $cloud := .Site.Params.cloudinary_cloud }}
{{ $template := .Site.Params.cloudinary_template }}
{{ $title := .Title | urlquery }}
{{ $section := .Section | urlquery }}

{{ $ogURL := printf "https://res.cloudinary.com/%s/image/upload/w_1200,h_630,c_fill,q_auto,f_auto/l_text:Inter_72_bold:%s,co_rgb:FFFFFF,g_south_west,x_80,y_200/l_text:Inter_36:%s,co_rgb:AAAAAA,g_south_west,x_80,y_120/%s" $cloud $title $section $template }}

<meta property="og:image" content="{{ $ogURL }}">
```

**Cloudinary transformations:**

```
w_1200,h_630      - Width and height
c_fill             - Crop to fill
q_auto             - Auto quality
f_auto             - Auto format
l_text:Font_Size   - Text overlay
co_rgb:COLOR       - Text color
g_south_west       - Gravity (position)
x_80,y_200         - Offset from gravity
```

### imgix

**URL-based transformations:**

```go-html-template
{{ $baseURL := "https://yoursite.imgix.net" }}
{{ $imagePath := .Params.hero_image | default "/images/default-og.jpg" }}
{{ $title := .Title | urlquery }}

{{ $ogURL := printf "%s%s?w=1200&h=630&fit=crop&txt=%s&txt-color=fff&txt-size=72&txt-align=middle,center&txt-font=Avenir+Next+Bold" $baseURL $imagePath $title }}

<meta property="og:image" content="{{ $ogURL }}">
```

## Build-Time Generation

### Node.js Script

**scripts/generate-og-images.js:**

```javascript
const fs = require('fs');
const path = require('path');
const puppeteer = require('puppeteer');
const matter = require('gray-matter');
const glob = require('glob');

async function generateOGImages() {
  const browser = await puppeteer.launch({ headless: 'new' });
  const page = await browser.newPage();
  await page.setViewport({ width: 1200, height: 630 });

  // Find all content files
  const files = glob.sync('content/**/*.md');

  for (const file of files) {
    const content = fs.readFileSync(file, 'utf8');
    const { data } = matter(content);

    if (!data.title) continue;

    const slug = path.basename(file, '.md');
    const outputDir = path.join('static', 'images', 'og');
    const outputPath = path.join(outputDir, `${slug}.jpg`);

    // Skip if already exists
    if (fs.existsSync(outputPath)) continue;

    // Ensure output directory exists
    fs.mkdirSync(outputDir, { recursive: true });

    // Generate HTML template
    const html = `
    <!DOCTYPE html>
    <html>
    <head>
      <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
          width: 1200px;
          height: 630px;
          display: flex;
          flex-direction: column;
          justify-content: center;
          padding: 80px;
          background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
          font-family: system-ui, -apple-system, sans-serif;
        }
        h1 {
          color: #ffffff;
          font-size: ${data.title.length > 60 ? '56' : '72'}px;
          font-weight: 800;
          line-height: 1.2;
          margin-bottom: 30px;
          max-width: 900px;
        }
        .meta {
          color: #8899aa;
          font-size: 28px;
        }
        .brand {
          position: absolute;
          bottom: 40px;
          right: 60px;
          color: #4488cc;
          font-size: 24px;
          font-weight: 600;
        }
      </style>
    </head>
    <body>
      <h1>${data.title}</h1>
      <div class="meta">${data.date ? new Date(data.date).toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' }) : ''}</div>
      <div class="brand">Hugo Best Practices</div>
    </body>
    </html>`;

    await page.setContent(html);
    await page.screenshot({
      path: outputPath,
      type: 'jpeg',
      quality: 85
    });

    console.log(`Generated: ${outputPath}`);
  }

  await browser.close();
}

generateOGImages().catch(console.error);
```

**Run before Hugo build:**

```bash
node scripts/generate-og-images.js && hugo
```

### GitHub Actions Integration

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-node@v3
        with:
          node-version: 20

      - name: Install dependencies
        run: npm install puppeteer gray-matter glob

      - name: Generate OG Images
        run: node scripts/generate-og-images.js

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'

      - name: Build
        run: hugo --minify

      - name: Deploy
        # Your deployment step
```

## Design Templates

### Professional Blog Post

**Layout:**

```
┌─────────────────────────────────────────┐
│                                         │
│    [Category Tag]                       │
│                                         │
│    Main Headline Text                   │
│    That Can Span Two Lines              │
│                                         │
│    Author Name  •  Date                 │
│                                         │
│                          [Brand Logo]   │
└─────────────────────────────────────────┘
```

### Tutorial/Guide

```
┌─────────────────────────────────────────┐
│  ┌──────────┐                           │
│  │  STEP BY │                           │
│  │   STEP   │  Tutorial Title           │
│  └──────────┘  Goes Here                │
│                                         │
│  ── ── ── ── ── ── ── ── ── ── ── ──   │
│                                         │
│  Subtitle or description text           │
│                          [Brand Logo]   │
└─────────────────────────────────────────┘
```

### Minimal/Clean

```
┌─────────────────────────────────────────┐
│                                         │
│                                         │
│         Article Title Here              │
│                                         │
│         sitename.com                    │
│                                         │
│                                         │
└─────────────────────────────────────────┘
```

### Color Schemes

**Professional:**
- Background: #1a1a2e → #16213e gradient
- Title: #ffffff
- Subtitle: #8899aa
- Accent: #4488cc

**Vibrant:**
- Background: #ff6b35 → #f7c59f gradient
- Title: #ffffff
- Subtitle: #fff8f0
- Accent: #ffffff

**Dark:**
- Background: #000000 → #1a1a1a gradient
- Title: #ffffff
- Subtitle: #666666
- Accent: #00d4ff

## Front Matter Configuration

### Basic

```yaml
---
title: "10 Hugo Performance Tips"
description: "Optimize your Hugo site for speed"
images:
  - /images/blog/hero.jpg
---
```

### With OG Image Override

```yaml
---
title: "10 Hugo Performance Tips"
og_image: /images/og/hugo-performance.jpg
twitter_image: /images/twitter/hugo-performance.jpg
---
```

### With Generation Parameters

```yaml
---
title: "10 Hugo Performance Tips"
og_template: "blog"
og_bg_color: "#1a1a2e"
og_text_color: "#ffffff"
og_subtitle: "Step-by-step guide"
---
```

## Best Practices

### Design

**✅ DO:**
- Use 1200×630 pixels (universal size)
- Include readable text (60px+ for headlines)
- Add brand logo/name
- Use high-contrast colors
- Test at mobile display size (345×181)
- Keep important content centered
- Leave safe margins (40-60px)
- Use consistent branding

**❌ DON'T:**
- Use small, illegible text
- Include too much text (< 20% of area)
- Use low-contrast colors
- Place critical info near edges
- Use white backgrounds
- Forget brand identity

### Performance

**✅ DO:**
- Optimize file size (< 500KB)
- Use JPEG for photos (q85)
- Use PNG for graphics with text
- Cache generated images
- Skip regeneration for unchanged content
- Use Hugo image processing when possible

**❌ DON'T:**
- Generate images on every build (cache them)
- Use unoptimized images (> 1MB)
- Regenerate unchanged images
- Block build on image generation failures

### Automation

**✅ DO:**
- Automate for all content types
- Provide fallback images
- Test generated images regularly
- Include in CI/CD pipeline
- Cache between builds

**❌ DON'T:**
- Rely on manual creation
- Skip fallback images
- Ignore generation failures
- Block deploys on image issues

## Guidelines

### Essential

**Minimum image setup:**
- Default OG image for site
- Hugo image processing for page images
- Consistent 1200×630 dimensions
- Fallback image chain

### Recommended

**For better social sharing:**
- Category-specific default images
- Hugo Pipes image processing
- Brand-consistent templates
- Both OG and Twitter images
- Alt text for accessibility

### Advanced

**For maximum automation:**
- Build-time generation with Puppeteer
- External service integration (Cloudinary)
- Dynamic text overlays
- Per-page customization
- A/B testing different designs

## Benefits

Consistency. Every page has a branded social image.

Efficiency. Zero manual effort per page after setup.

Branding. Professional appearance across all platforms.

Engagement. Custom images increase click-through rates.

Scalability. Works for 10 or 10,000 pages.

## Related

- [open-graph-meta-tags.md](./open-graph-meta-tags.md) - Open Graph image requirements
- [twitter-card-images.md](./twitter-card-images.md) - Twitter Card image specifications
- [linkedin-specific-meta.md](./linkedin-specific-meta.md) - LinkedIn image requirements
- [pinterest-rich-pins.md](./pinterest-rich-pins.md) - Pinterest image specifications
- [../../01-hugo-basics/hugo-asset-pipeline.md](../01-hugo-basics/hugo-asset-pipeline.md) - Hugo Pipes image processing
