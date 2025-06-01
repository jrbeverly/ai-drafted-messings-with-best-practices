# Meta Descriptions and Titles

Meta title tags. Meta descriptions. SERP optimization. Click-through rate. Character limits. Hugo implementation.

## Principle

Optimize meta titles and descriptions for search engine results pages. Write compelling copy that increases click-through rates. Follow character limits for proper display. Use Hugo templates for consistent, automated generation.

## Meta Title Tag

**The most important on-page SEO element.**

### What is the Title Tag?

**HTML element:** `<title>Page Title</title>`

**Purpose:**
- Displayed as clickable headline in search results
- Shown in browser tab
- Used as default text when bookmarking
- Primary ranking signal for search engines

### Syntax

```html
<title>10 Essential Hugo Performance Tips | Hugo Best Practices</title>
```

### Character Limits

**Recommended:** 50-60 characters

**Google display:**
- Desktop: ~600 pixels wide (approximately 60 characters)
- Mobile: ~560 pixels wide (approximately 55 characters)

**If too long:**
- Google truncates with ellipsis (...)
- Important keywords may be cut off
- Looks unprofessional

**If too short:**
- Missed opportunity for keywords
- Less compelling in search results
- May appear thin or low-quality

### Title Tag Formulas

**Blog posts:**

```
[Primary Keyword]: [Benefit/Promise] | [Brand]
```

**Examples:**

```
Hugo Performance: 10 Tips to Load 3x Faster | Hugo Best Practices
Hugo Modules: Complete Guide for 2026 | Hugo Best Practices
Hugo vs Jekyll: Which Static Site Generator Wins? | Hugo Best
```

**Product pages:**

```
[Product Name] - [Key Benefit] | [Brand]
```

**Homepage:**

```
[Brand Name] - [Primary Value Proposition]
```

**Category/list pages:**

```
[Category] Articles & Tutorials | [Brand]
```

### Title Tag Best Practices

**✅ DO:**
- Include primary keyword near the beginning
- Keep under 60 characters
- Make it compelling and clickable
- Include brand name (end of title)
- Use separator (| or - or :)
- Unique title for every page
- Match search intent
- Front-load important words

**❌ DON'T:**
- Stuff keywords
- Use all caps
- Duplicate titles across pages
- Use generic titles ("Home", "Page 1")
- Start with brand name (waste of space)
- Use special characters excessively
- Exceed 60 characters for important words

### Title Tag Patterns

**Pattern 1: Keyword | Brand**

```
Hugo Performance Tips | Hugo Best Practices
```

**Pattern 2: Keyword - Description | Brand**

```
Hugo Modules - Complete Setup Guide | Hugo Best Practices
```

**Pattern 3: Number + Keyword + Benefit**

```
10 Hugo Tips That Will Make Your Site 3x Faster
```

**Pattern 4: How to + Keyword**

```
How to Optimize Hugo Build Times in 2026
```

**Pattern 5: Question format**

```
Is Hugo the Fastest Static Site Generator? | Hugo Best
```

## Meta Description

**The sales pitch for your page in search results.**

### What is the Meta Description?

**HTML element:**

```html
<meta name="description" content="Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples and benchmarks.">
```

**Purpose:**
- Displayed as snippet text under title in search results
- Influences click-through rate (CTR)
- NOT a direct ranking factor (but CTR is)
- Google may override with page content

### Character Limits

**Recommended:** 120-160 characters

**Google display:**
- Desktop: ~920 pixels (approximately 155-160 characters)
- Mobile: ~680 pixels (approximately 120 characters)

**Target:** Write for 155 characters, front-load important info in first 120

### Description Formulas

**Informational content:**

```
[What you'll learn]. [Key benefit]. [Call to action].
```

**Example:**

```
Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples. Start optimizing today.
```

**Product/service:**

```
[What it is]. [Key benefit]. [Social proof/urgency].
```

**Example:**

```
Professional Hugo theme with 50+ components and dark mode. Trusted by 5,000+ developers. Free updates for life.
```

**Tutorial/how-to:**

```
[Step-by-step guide to X]. [What you'll achieve]. [Difficulty/time].
```

**Example:**

```
Step-by-step guide to deploying Hugo sites on Netlify. Get your site live in under 5 minutes. Beginner-friendly with screenshots.
```

### Description Best Practices

**✅ DO:**
- Include primary keyword naturally
- Write compelling copy (it's an ad)
- Include call to action
- Front-load important information
- Keep between 120-160 characters
- Unique description for every page
- Match search intent
- Include numbers and specifics
- Use active voice

**❌ DON'T:**
- Stuff keywords
- Use generic descriptions
- Duplicate across pages
- Exceed 160 characters for key info
- Use quotes (Google may trim)
- Start with "This page is about..."
- Use first paragraph of content as-is
- Leave empty (Google generates one)

### Writing Compelling Descriptions

**Power words that increase CTR:**

```
Learn, Discover, Master, Complete, Ultimate, Essential,
Proven, Step-by-step, Free, Quick, Easy, Beginner,
Advanced, Expert, 2026, Updated, Guide, Tutorial,
Tips, Tricks, Secrets, Boost, Optimize, Fast
```

**Include specifics:**

```
❌ VAGUE: "Learn about Hugo performance"
✅ SPECIFIC: "Learn 10 proven techniques to cut Hugo build times by 80%"
```

**Include social proof:**

```
❌ GENERIC: "A good Hugo theme"
✅ SOCIAL PROOF: "Hugo theme trusted by 5,000+ developers. 4.8/5 rating."
```

**Include urgency/timeliness:**

```
❌ DATED: "Hugo tips"
✅ TIMELY: "Updated for Hugo 0.120+ (2026). Latest performance techniques."
```

## Hugo Implementation

### Basic Title

**layouts/_default/baseof.html:**

```go-html-template
<head>
  <title>{{ if .IsHome }}{{ .Site.Title }}{{ else }}{{ .Title }} | {{ .Site.Title }}{{ end }}</title>
</head>
```

**Output:**
- Homepage: `Hugo Best Practices`
- Blog post: `10 Hugo Performance Tips | Hugo Best Practices`

### Advanced Title Template

**layouts/partials/head/title.html:**

```go-html-template
{{- $title := "" -}}

{{- if .IsHome -}}
  {{- $title = .Site.Title -}}
  {{- with .Site.Params.tagline -}}
    {{- $title = printf "%s - %s" $.Site.Title . -}}
  {{- end -}}

{{- else if .Params.seo_title -}}
  {{/* Custom SEO title from front matter */}}
  {{- $title = .Params.seo_title -}}

{{- else if eq .Kind "taxonomy" -}}
  {{/* Taxonomy page (tags, categories) */}}
  {{- $title = printf "%s Articles | %s" .Title .Site.Title -}}

{{- else if eq .Kind "term" -}}
  {{/* Individual term page */}}
  {{- $title = printf "%s: %s | %s" (humanize .Data.Singular) .Title .Site.Title -}}

{{- else -}}
  {{/* Regular pages */}}
  {{- $title = printf "%s | %s" .Title .Site.Title -}}

{{- end -}}

<title>{{ $title }}</title>
```

### Basic Description

**layouts/partials/head/description.html:**

```go-html-template
{{ $description := "" }}

{{ if .Params.description }}
  {{ $description = .Params.description }}
{{ else if .IsHome }}
  {{ $description = .Site.Params.description }}
{{ else if .Summary }}
  {{ $description = .Summary | plainify | truncate 155 }}
{{ else }}
  {{ $description = .Site.Params.description }}
{{ end }}

{{ with $description }}
  <meta name="description" content="{{ . }}">
{{ end }}
```

### Advanced Description Template

**layouts/partials/head/description.html:**

```go-html-template
{{- $description := "" -}}

{{- if .Params.seo_description -}}
  {{/* Custom SEO description from front matter */}}
  {{- $description = .Params.seo_description -}}

{{- else if .Params.description -}}
  {{/* Front matter description */}}
  {{- $description = .Params.description -}}

{{- else if .IsHome -}}
  {{/* Homepage */}}
  {{- $description = .Site.Params.description -}}

{{- else if eq .Kind "taxonomy" -}}
  {{/* Taxonomy page */}}
  {{- $description = printf "Browse all %s articles and tutorials on %s." .Title .Site.Title -}}

{{- else if eq .Kind "term" -}}
  {{/* Term page */}}
  {{- $count := len .Pages -}}
  {{- $description = printf "Explore %d articles tagged with %s on %s." $count .Title .Site.Title -}}

{{- else if .Summary -}}
  {{/* Auto-generate from content summary */}}
  {{- $description = .Summary | plainify | replaceRE "\\s+" " " | truncate 155 -}}

{{- else -}}
  {{/* Fallback to site description */}}
  {{- $description = .Site.Params.description -}}

{{- end -}}

{{- with $description -}}
  <meta name="description" content="{{ . }}">
{{- end -}}
```

### Complete Head Template

**layouts/partials/head/seo.html:**

```go-html-template
{{/* Title */}}
{{ partial "head/title.html" . }}

{{/* Meta description */}}
{{ partial "head/description.html" . }}

{{/* Canonical URL */}}
{{ partial "head/canonical.html" . }}

{{/* Open Graph */}}
{{ partial "head/opengraph.html" . }}

{{/* Twitter Card */}}
{{ partial "head/twitter-card.html" . }}

{{/* Hreflang */}}
{{ partial "head/hreflang.html" . }}

{{/* Structured Data */}}
{{ if .IsHome }}
  {{ partial "structured-data/website.html" . }}
{{ else if .IsPage }}
  {{ partial "structured-data/article.html" . }}
{{ end }}
{{ partial "structured-data/breadcrumb.html" . }}
```

**Include in baseof.html:**

```go-html-template
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  {{ partial "head/seo.html" . }}
</head>
```

### Content Front Matter

**Standard front matter:**

```yaml
---
title: "10 Essential Hugo Performance Tips"
description: "Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples."
date: 2026-02-13T10:00:00Z
---
```

**With custom SEO overrides:**

```yaml
---
title: "10 Essential Hugo Performance Tips for Lightning-Fast Static Sites"
description: "Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples."

# SEO overrides (shorter/different from display title)
seo_title: "10 Hugo Performance Tips | 3x Faster Sites"
seo_description: "Boost Hugo site speed by 80%. 10 proven optimization techniques with code examples. Updated for Hugo 0.120+ (2026)."
---
```

**Why separate SEO fields?**
- Display title can be longer/different from SEO title
- Description can differ from meta description
- Allows A/B testing without changing page content

### Site Configuration

**config.toml:**

```toml
title = "Hugo Best Practices"

[params]
  description = "Learn to build fast, secure static sites with Hugo. Tutorials, best practices, and production-ready templates."
  tagline = "Build Fast Static Sites"

  # Separator for title (brand separator)
  titleSeparator = "|"
```

## SERP Preview

### Google Search Result Format

**Desktop:**

```
Title Tag (up to ~60 characters)
https://example.com/blog/hugo-tips/
Meta description text here, up to approximately 155-160 characters.
This is the snippet that appears under the title in search results.
```

**Mobile:**

```
Title Tag (up to ~55 characters)
example.com › blog › hugo-tips
Meta description up to approximately 120 characters.
Shorter on mobile devices.
```

### Optimal Lengths

| Element | Min | Target | Max |
|---------|-----|--------|-----|
| Title tag | 30 | 55 | 60 |
| Meta description | 70 | 140 | 160 |
| URL slug | 3 | 5-7 | 15 words |

### Character Count in Hugo

**Auto-truncate long descriptions:**

```go-html-template
{{ $description := .Description | truncate 155 }}
<meta name="description" content="{{ $description }}">
```

**Count characters in front matter (manual check):**

```bash
# Check title lengths across all content
find content/ -name "*.md" -exec grep -l "title:" {} \; | while read f; do
  title=$(grep "^title:" "$f" | head -1 | sed 's/title: *"//;s/"$//')
  len=${#title}
  if [ $len -gt 60 ]; then
    echo "⚠️ $f: $len chars - $title"
  fi
done
```

## Testing and Validation

### SERP Preview Tools

**Online tools:**
- Google SERP Simulator
- Moz Title Tag Preview Tool
- SEOmofo SERP Snippet Optimizer

### Google Search Console

**Performance Report:**
1. Go to Google Search Console
2. Click "Performance"
3. View CTR for different pages
4. Identify low-CTR pages
5. Improve titles and descriptions

**URL Inspection:**
1. Enter page URL
2. View "Page title" and "Meta description"
3. See how Google renders them

### Manual Testing

**Check title and description:**

```bash
# Check title
curl -s https://example.com/blog/post/ | grep -o '<title>[^<]*</title>'

# Check meta description
curl -s https://example.com/blog/post/ | grep 'name="description"'
```

**Check character count:**

```bash
# Title length
curl -s https://example.com/blog/post/ | grep -o '<title>[^<]*</title>' | sed 's/<[^>]*>//g' | wc -c

# Description length
curl -s https://example.com/blog/post/ | grep -o 'content="[^"]*"' | head -1 | sed 's/content="//;s/"$//' | wc -c
```

### Hugo Build Check

```bash
hugo

# Check generated titles
grep -r '<title>' public/ | head -20

# Check generated descriptions
grep -r 'name="description"' public/ | head -20
```

## Common Issues

### Title Too Long

**Problem:** Title truncated in search results

**Solution:**

```yaml
# ❌ TOO LONG (85 characters)
title: "The Complete Beginner's Guide to Hugo Static Site Generator Performance Optimization Tips"

# ✅ OPTIMIZED (52 characters)
title: "Hugo Performance: 10 Tips for 3x Faster Sites"
```

### Missing Meta Description

**Problem:** Google generates description from page content

**Impact:**
- Unpredictable snippet text
- May not be compelling
- Missed optimization opportunity

**Solution:**

Always provide description:

```yaml
---
title: "Hugo Performance Tips"
description: "Learn 10 proven techniques to make your Hugo site load 3x faster. Step-by-step guide with code examples."
---
```

### Duplicate Titles

**Problem:** Multiple pages with same title

**Impact:**
- Search engines confused
- Potential keyword cannibalization
- Poor user experience

**Solution:**

Unique title for every page:

```yaml
# Page 1
title: "Hugo Performance Tips for Build Speed"

# Page 2
title: "Hugo Performance Tips for Page Load Speed"

# Page 3
title: "Hugo Performance Tips for Image Optimization"
```

### Duplicate Descriptions

**Problem:** Same description on multiple pages

**Solution:**

Unique description for each page:

```go-html-template
{{ if .Params.description }}
  <meta name="description" content="{{ .Params.description }}">
{{ else }}
  {{/* Auto-generate from content */}}
  <meta name="description" content="{{ .Summary | plainify | truncate 155 }}">
{{ end }}
```

### Google Overrides Your Description

**Problem:** Google shows different description than meta tag

**Why:** Google thinks page content is more relevant to search query

**Solutions:**
- Write descriptions that match search intent
- Include keywords users search for
- Make description accurate to page content
- Can't force Google to use your description

## Best Practices

### Title Tags

**✅ DO:**
- Include primary keyword near start
- Keep under 60 characters
- Unique per page
- Include brand name at end
- Use separator (| or -)
- Write for humans (compelling)
- Match search intent
- Use numbers when possible

**❌ DON'T:**
- Keyword stuff ("Hugo Hugo Hugo Tips")
- Use all caps
- Start with brand name
- Use generic titles ("Welcome")
- Duplicate across pages
- Exceed 60 characters for key content

### Meta Descriptions

**✅ DO:**
- Include primary keyword naturally
- Write compelling copy (it's an ad)
- Include call to action
- Keep 120-160 characters
- Front-load important info
- Use active voice
- Include numbers/specifics
- Unique per page

**❌ DON'T:**
- Keyword stuff
- Use generic copy
- Exceed 160 characters for key info
- Duplicate across pages
- Start with "This page..."
- Leave empty
- Use quotes (may trim)
- Copy first paragraph verbatim

### Hugo Implementation

**✅ DO:**
- Provide description in front matter
- Use SEO overrides for different display vs SEO
- Auto-generate fallback from content
- Truncate to safe character limits
- Test generated output

**❌ DON'T:**
- Rely on auto-generation alone
- Skip front matter descriptions
- Forget to test character lengths
- Use same template for all page types

## Guidelines

### Essential

**Every page must have:**
- `<title>` tag (unique, under 60 characters)
- `<meta name="description">` (unique, 120-160 characters)
- Primary keyword in both
- Brand name in title

### Recommended

**For better CTR:**
- Custom SEO title/description fields
- Numbers in title (10 tips, 5 ways)
- Call to action in description
- Active voice
- Power words
- Match search intent

### Advanced

**For maximum optimization:**
- A/B test titles via Google Search Console
- Monitor CTR by page
- Seasonal title updates
- Dynamic descriptions per content type
- SERP preview tool validation

## Benefits

Higher CTR. Compelling titles and descriptions increase click-through rates.

Better Rankings. Title tags are a primary ranking signal.

User Experience. Users find exactly what they're looking for.

Brand Visibility. Consistent brand presence in search results.

Content Discovery. Well-written descriptions attract more visitors.

## Related

- [meta-tags-comprehensive.md](./meta-tags-comprehensive.md) - All HTML meta tags
- [open-graph-meta-tags.md](./open-graph-meta-tags.md) - Social media title/description
- [twitter-card-types.md](./twitter-card-types.md) - Twitter Card title/description
- [structured-data-schema-org.md](./structured-data-schema-org.md) - Schema.org headline
- [canonical-urls.md](./canonical-urls.md) - Canonical URL for proper indexing
