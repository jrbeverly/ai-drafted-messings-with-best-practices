# Hreflang Tags

Hreflang attributes. Multilingual SEO. International targeting. Language and region targeting. x-default hreflang.

## Principle

Use hreflang tags to indicate language and regional variations of your content. Help search engines serve the correct language version to users. Prevent duplicate content issues for multilingual sites. Improve international SEO.

## What are Hreflang Tags?

**Hreflang:** HTML attribute indicating language and geographical targeting

**Purpose:** Tell search engines which language/region version to show users

**Format:** `<link rel="alternate" hreflang="lang-REGION" href="URL">`

**Example:**

```html
<link rel="alternate" hreflang="en" href="https://example.com/en/">
<link rel="alternate" hreflang="es" href="https://example.com/es/">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/">
```

**Specification:** https://developers.google.com/search/docs/specialty/international/localized-versions

## Why Use Hreflang?

### International SEO Benefits

**Without hreflang:**
- Google may show wrong language to users
- Duplicate content issues (English UK vs English US)
- Poor user experience (Spanish user gets English page)
- Lost international traffic

**With hreflang:**
- ✅ Correct language served to users
- ✅ No duplicate content penalties
- ✅ Better user experience
- ✅ Improved international rankings
- ✅ Regional targeting (en-US vs en-GB)

### Use Cases

**Multilingual sites:**
- English, Spanish, French, German versions
- Each language on same domain (example.com/en/, example.com/es/)

**Multi-regional sites:**
- Same language, different regions
- English for US, UK, Canada, Australia (en-US, en-GB, en-CA, en-AU)

**Combination:**
- Spanish for Spain and Mexico (es-ES, es-MX)
- French for France and Canada (fr-FR, fr-CA)

## Hreflang Syntax

### Language Code

**Format:** ISO 639-1 (two-letter language code)

**Examples:**

```html
<link rel="alternate" hreflang="en" href="https://example.com/en/">
<link rel="alternate" hreflang="es" href="https://example.com/es/">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/">
<link rel="alternate" hreflang="de" href="https://example.com/de/">
<link rel="alternate" hreflang="zh" href="https://example.com/zh/">
<link rel="alternate" hreflang="ja" href="https://example.com/ja/">
```

**Common language codes:**

| Code | Language |
|------|----------|
| en | English |
| es | Spanish |
| fr | French |
| de | German |
| it | Italian |
| pt | Portuguese |
| ru | Russian |
| zh | Chinese |
| ja | Japanese |
| ko | Korean |
| ar | Arabic |
| hi | Hindi |
| nl | Dutch |
| sv | Swedish |
| pl | Polish |

### Language + Region Code

**Format:** ISO 639-1 + ISO 3166-1 (language-REGION)

**Examples:**

```html
<!-- English variations -->
<link rel="alternate" hreflang="en-US" href="https://example.com/en-us/">
<link rel="alternate" hreflang="en-GB" href="https://example.com/en-gb/">
<link rel="alternate" hreflang="en-CA" href="https://example.com/en-ca/">
<link rel="alternate" hreflang="en-AU" href="https://example.com/en-au/">

<!-- Spanish variations -->
<link rel="alternate" hreflang="es-ES" href="https://example.com/es-es/">
<link rel="alternate" hreflang="es-MX" href="https://example.com/es-mx/">
<link rel="alternate" hreflang="es-AR" href="https://example.com/es-ar/">

<!-- French variations -->
<link rel="alternate" hreflang="fr-FR" href="https://example.com/fr-fr/">
<link rel="alternate" hreflang="fr-CA" href="https://example.com/fr-ca/">
```

**Common region codes:**

| Code | Country/Region |
|------|----------------|
| US | United States |
| GB | United Kingdom |
| CA | Canada |
| AU | Australia |
| ES | Spain |
| MX | Mexico |
| AR | Argentina |
| FR | France |
| DE | Germany |
| IT | Italy |
| BR | Brazil |
| CN | China |
| JP | Japan |
| KR | South Korea |
| IN | India |

### x-default

**Fallback for unmatched languages:**

```html
<link rel="alternate" hreflang="x-default" href="https://example.com/en/">
```

**Use x-default for:**
- Default language when user's language not available
- Language selector page
- Automatic redirect page

**Example:**

```html
<!-- x-default points to English (default) -->
<link rel="alternate" hreflang="x-default" href="https://example.com/en/">
<link rel="alternate" hreflang="en" href="https://example.com/en/">
<link rel="alternate" hreflang="es" href="https://example.com/es/">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/">
```

## Hreflang Implementation

### Basic Structure

**Each language version includes all hreflang tags:**

**English page (example.com/en/):**

```html
<head>
  <!-- Self-referencing hreflang -->
  <link rel="alternate" hreflang="en" href="https://example.com/en/">
  <!-- Alternate languages -->
  <link rel="alternate" hreflang="es" href="https://example.com/es/">
  <link rel="alternate" hreflang="fr" href="https://example.com/fr/">
  <!-- Default -->
  <link rel="alternate" hreflang="x-default" href="https://example.com/en/">
</head>
```

**Spanish page (example.com/es/):**

```html
<head>
  <!-- Self-referencing hreflang -->
  <link rel="alternate" hreflang="es" href="https://example.com/es/">
  <!-- Alternate languages -->
  <link rel="alternate" hreflang="en" href="https://example.com/en/">
  <link rel="alternate" hreflang="fr" href="https://example.com/fr/">
  <!-- Default -->
  <link rel="alternate" hreflang="x-default" href="https://example.com/en/">
</head>
```

**French page (example.com/fr/):**

```html
<head>
  <!-- Self-referencing hreflang -->
  <link rel="alternate" hreflang="fr" href="https://example.com/fr/">
  <!-- Alternate languages -->
  <link rel="alternate" hreflang="en" href="https://example.com/en/">
  <link rel="alternate" hreflang="es" href="https://example.com/es/">
  <!-- Default -->
  <link rel="alternate" hreflang="x-default" href="https://example.com/en/">
</head>
```

**Critical rule:** All pages must reference each other bidirectionally.

## Hugo Implementation

### Hugo Multilingual Configuration

**config.toml:**

```toml
defaultContentLanguage = "en"
defaultContentLanguageInSubdir = true

[languages]
  [languages.en]
    languageName = "English"
    weight = 1
    contentDir = "content/en"

  [languages.es]
    languageName = "Español"
    weight = 2
    contentDir = "content/es"

  [languages.fr]
    languageName = "Français"
    weight = 3
    contentDir = "content/fr"
```

**Content structure:**

```
content/
├── en/
│   └── blog/
│       └── post.md
├── es/
│   └── blog/
│       └── post.md
└── fr/
    └── blog/
        └── post.md
```

### Automatic Hreflang Template

**Hugo automatically provides translations via `.Translations`:**

**layouts/partials/head/hreflang.html:**

```go-html-template
{{/* Self-referencing hreflang */}}
<link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">

{{/* Alternate language versions */}}
{{ range .Translations }}
  <link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">
{{ end }}

{{/* x-default (use default language) */}}
{{ if .IsTranslated }}
  {{ with .Sites.First.Home }}
    <link rel="alternate" hreflang="x-default" href="{{ .Permalink }}">
  {{ end }}
{{ end }}
```

### Regional Hreflang (Language + Region)

**config.toml:**

```toml
[languages]
  [languages.en-US]
    languageName = "English (US)"
    weight = 1

  [languages.en-GB]
    languageName = "English (UK)"
    weight = 2

  [languages.es-ES]
    languageName = "Español (España)"
    weight = 3

  [languages.es-MX]
    languageName = "Español (México)"
    weight = 4
```

**Template:**

```go-html-template
{{/* Hugo automatically uses language code from config */}}
<link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">

{{ range .Translations }}
  <link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">
{{ end }}
```

**Hugo will output:**

```html
<link rel="alternate" hreflang="en-US" href="https://example.com/en-us/">
<link rel="alternate" hreflang="en-GB" href="https://example.com/en-gb/">
<link rel="alternate" hreflang="es-ES" href="https://example.com/es-es/">
<link rel="alternate" hreflang="es-MX" href="https://example.com/es-mx/">
```

### Custom x-default

**Set x-default to language selector or primary language:**

```go-html-template
{{/* Current language */}}
<link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">

{{/* All translations */}}
{{ range .Translations }}
  <link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">
{{ end }}

{{/* x-default: Use site's default language */}}
{{ $defaultLang := .Site.Language.Lang }}
{{ if eq .Language.Lang $defaultLang }}
  <link rel="alternate" hreflang="x-default" href="{{ .Permalink }}">
{{ else }}
  {{ with .Translations.ByLanguage (index .Site.Languages 0) }}
    <link rel="alternate" hreflang="x-default" href="{{ .Permalink }}">
  {{ end }}
{{ end }}
```

### Include in Layout

**layouts/_default/baseof.html:**

```go-html-template
<!DOCTYPE html>
<html lang="{{ .Language.Lang }}">
<head>
  <meta charset="utf-8">
  <title>{{ .Title }}</title>

  {{/* Canonical */}}
  <link rel="canonical" href="{{ .Permalink }}">

  {{/* Hreflang */}}
  {{ partial "head/hreflang.html" . }}

  {{/* Other head elements */}}
</head>
<body>
  {{ block "main" . }}{{ end }}
</body>
</html>
```

## Advanced Patterns

### Complete Multilingual Template

**layouts/partials/head/hreflang.html:**

```go-html-template
{{/* Hreflang tags for multilingual SEO */}}

{{/* Self-referencing hreflang */}}
<link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">

{{/* Alternate language versions */}}
{{ if .IsTranslated }}
  {{ range .Translations }}
    <link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">
  {{ end }}
{{ end }}

{{/* x-default fallback */}}
{{ if .IsTranslated }}
  {{/* Use default content language for x-default */}}
  {{ $defaultLang := .Site.Language.Lang }}
  {{ if eq .Language.Lang $defaultLang }}
    <link rel="alternate" hreflang="x-default" href="{{ .Permalink }}">
  {{ else }}
    {{/* Find default language version */}}
    {{ range .AllTranslations }}
      {{ if eq .Language.Lang $defaultLang }}
        <link rel="alternate" hreflang="x-default" href="{{ .Permalink }}">
      {{ end }}
    {{ end }}
  {{ end }}
{{ else }}
  {{/* Not translated - this is the only version */}}
  <link rel="alternate" hreflang="x-default" href="{{ .Permalink }}">
{{ end }}
```

### Sitemap with Hreflang

**Hugo automatically includes hreflang in sitemaps for multilingual sites:**

**Generated sitemap:**

```xml
<url>
  <loc>https://example.com/en/blog/post/</loc>
  <xhtml:link rel="alternate" hreflang="en" href="https://example.com/en/blog/post/"/>
  <xhtml:link rel="alternate" hreflang="es" href="https://example.com/es/blog/post/"/>
  <xhtml:link rel="alternate" hreflang="fr" href="https://example.com/fr/blog/post/"/>
</url>
```

**Custom sitemap template with hreflang:**

**layouts/sitemap.xml:**

```xml
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
  xmlns:xhtml="http://www.w3.org/1999/xhtml">
  {{ range .Data.Pages }}
  <url>
    <loc>{{ .Permalink }}</loc>
    {{ if not .Lastmod.IsZero }}
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
    {{ end }}

    {{/* Hreflang in sitemap */}}
    <xhtml:link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}"/>
    {{ range .Translations }}
    <xhtml:link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}"/>
    {{ end }}
  </url>
  {{ end }}
</urlset>
```

### Language Switcher

**Complement hreflang with visible language switcher:**

**layouts/partials/language-switcher.html:**

```go-html-template
<div class="language-switcher">
  {{ if .IsTranslated }}
    <ul>
      {{/* Current language */}}
      <li class="active">{{ .Language.LanguageName }}</li>

      {{/* Available translations */}}
      {{ range .Translations }}
        <li>
          <a href="{{ .Permalink }}" hreflang="{{ .Language.Lang }}">
            {{ .Language.LanguageName }}
          </a>
        </li>
      {{ end }}
    </ul>
  {{ end }}
</div>
```

## Hreflang vs Canonical

### Use Both Together

**Hreflang:** Indicates language/region alternatives

**Canonical:** Indicates preferred URL for duplicate content

**For translated content (unique):**

```html
<!-- English page -->
<link rel="canonical" href="https://example.com/en/page/">
<link rel="alternate" hreflang="en" href="https://example.com/en/page/">
<link rel="alternate" hreflang="es" href="https://example.com/es/page/">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/page/">
```

**Each language version:**
- Self-referencing canonical
- Hreflang to all translations

**For duplicate content (same language, different region):**

```html
<!-- English US (canonical) -->
<link rel="canonical" href="https://example.com/en-us/">
<link rel="alternate" hreflang="en-US" href="https://example.com/en-us/">
<link rel="alternate" hreflang="en-GB" href="https://example.com/en-gb/">

<!-- English GB (duplicate, points to US) -->
<link rel="canonical" href="https://example.com/en-us/">
<link rel="alternate" hreflang="en-US" href="https://example.com/en-us/">
<link rel="alternate" hreflang="en-GB" href="https://example.com/en-gb/">
```

## Best Practices

### General

**✅ DO:**
- Use self-referencing hreflang on every page
- Include all language variations
- Use bidirectional links (all pages reference each other)
- Use absolute URLs (https://example.com/page/)
- Include x-default for fallback
- Use ISO 639-1 language codes
- Use ISO 3166-1 region codes (if regional targeting)
- Include hreflang in sitemap

**❌ DON'T:**
- Use relative URLs
- Omit self-referencing hreflang
- Create one-way links (missing bidirectional)
- Mix up language and region codes
- Use non-standard codes
- Point hreflang to different content
- Use hreflang for completely different content

### Language Codes

**✅ DO:**
- Use lowercase for language (en, es, fr)
- Use uppercase for region (US, GB, MX)
- Format: en-US (lowercase-UPPERCASE)
- Use ISO standard codes

**❌ DON'T:**
- Mix case (EN-us, En-US)
- Make up codes (english, spanish)
- Use wrong codes (uk instead of en-GB)

### x-default

**✅ DO:**
- Point to primary language version
- Use for language selector page
- Include on all pages

**❌ DON'T:**
- Omit x-default
- Point to redirect
- Change x-default per page

### Bidirectional Linking

**✅ DO:**
- Each page references all translations
- Include self-referencing hreflang
- Maintain symmetry

**❌ DON'T:**
- One-way links only
- Orphaned language versions
- Inconsistent hreflang across translations

## Testing and Validation

### Google Search Console

**International Targeting Report:**
1. Go to Google Search Console
2. Check "International Targeting"
3. View hreflang errors
4. Fix errors and warnings

**Common errors:**
- No return tags (missing bidirectional links)
- Incorrect hreflang language code
- Multiple pages for same language
- hreflang to non-canonical

### Hreflang Tags Testing Tool

**URL:** https://www.aleydasolis.com/english/international-seo-tools/hreflang-tags-generator/

**Or use:**
- https://technicalseo.com/tools/hreflang/

### Manual Testing

**Check page source:**

```bash
curl -s https://example.com/en/ | grep 'hreflang'
```

**Expected output:**

```html
<link rel="alternate" hreflang="en" href="https://example.com/en/">
<link rel="alternate" hreflang="es" href="https://example.com/es/">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/">
<link rel="alternate" hreflang="x-default" href="https://example.com/en/">
```

**Verify bidirectional:**

```bash
# Check English page has Spanish hreflang
curl -s https://example.com/en/ | grep 'hreflang="es"'

# Check Spanish page has English hreflang
curl -s https://example.com/es/ | grep 'hreflang="en"'
```

### Screaming Frog SEO Spider

**Bulk hreflang audit:**
1. Crawl site
2. Go to "Hreflang" tab
3. Check for errors
4. Export for review

## Common Issues

### Missing Bidirectional Links

**Problem:**
- English page has hreflang to Spanish
- Spanish page missing hreflang to English

**Impact:** Google may ignore hreflang

**Solution:**

Ensure all pages reference each other:

```go-html-template
{{/* Always include self and all translations */}}
<link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">
{{ range .Translations }}
  <link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">
{{ end }}
```

### Wrong Language Code

**Problem:**

```html
<!-- ❌ WRONG -->
<link rel="alternate" hreflang="sp" href="https://example.com/es/">
<link rel="alternate" hreflang="english" href="https://example.com/en/">
```

**Solution:**

```html
<!-- ✅ CORRECT -->
<link rel="alternate" hreflang="es" href="https://example.com/es/">
<link rel="alternate" hreflang="en" href="https://example.com/en/">
```

Use ISO 639-1 codes.

### Missing Self-Referencing Hreflang

**Problem:**

```html
<!-- English page -->
<link rel="alternate" hreflang="es" href="https://example.com/es/">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/">
<!-- Missing hreflang="en" -->
```

**Impact:** Google may be confused

**Solution:**

```html
<!-- ✅ CORRECT -->
<link rel="alternate" hreflang="en" href="https://example.com/en/">
<link rel="alternate" hreflang="es" href="https://example.com/es/">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/">
```

### Hreflang to Non-Canonical

**Problem:**

```html
<!-- Page has canonical pointing elsewhere -->
<link rel="canonical" href="https://example.com/other-page/">

<!-- But hreflang points here -->
<link rel="alternate" hreflang="en" href="https://example.com/this-page/">
```

**Impact:** Google may ignore hreflang

**Solution:**

Hreflang and canonical should align:

```html
<link rel="canonical" href="https://example.com/this-page/">
<link rel="alternate" hreflang="en" href="https://example.com/this-page/">
```

### Relative URLs

**Problem:**

```html
<!-- ❌ WRONG -->
<link rel="alternate" hreflang="en" href="/en/">
<link rel="alternate" hreflang="es" href="/es/">
```

**Solution:**

```html
<!-- ✅ CORRECT -->
<link rel="alternate" hreflang="en" href="https://example.com/en/">
<link rel="alternate" hreflang="es" href="https://example.com/es/">
```

## Complete Example

**config.toml:**

```toml
baseURL = "https://example.com"
defaultContentLanguage = "en"
defaultContentLanguageInSubdir = true

[languages]
  [languages.en]
    languageName = "English"
    weight = 1
    title = "Hugo Best Practices"

  [languages.es]
    languageName = "Español"
    weight = 2
    title = "Hugo Mejores Prácticas"

  [languages.fr]
    languageName = "Français"
    weight = 3
    title = "Hugo Meilleures Pratiques"
```

**layouts/partials/head/hreflang.html:**

```go-html-template
{{/* Self-referencing */}}
<link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">

{{/* All translations */}}
{{ range .Translations }}
  <link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">
{{ end }}

{{/* x-default */}}
{{ if .IsTranslated }}
  {{ $defaultLang := .Site.Language.Lang }}
  {{ range .AllTranslations }}
    {{ if eq .Language.Lang $defaultLang }}
      <link rel="alternate" hreflang="x-default" href="{{ .Permalink }}">
    {{ end }}
  {{ end }}
{{ else }}
  <link rel="alternate" hreflang="x-default" href="{{ .Permalink }}">
{{ end }}
```

**Output (English page):**

```html
<link rel="alternate" hreflang="en" href="https://example.com/en/blog/post/">
<link rel="alternate" hreflang="es" href="https://example.com/es/blog/post/">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/blog/post/">
<link rel="alternate" hreflang="x-default" href="https://example.com/en/blog/post/">
```

## Guidelines

### Essential

**Minimum for multilingual sites:**
- Self-referencing hreflang on every page
- Bidirectional links to all translations
- Absolute URLs with HTTPS
- ISO standard language codes
- x-default for fallback

### Recommended

**For better international SEO:**
- Include hreflang in sitemap
- Regional codes where applicable (en-US, en-GB)
- Visible language switcher
- Test with Google Search Console
- Monitor international targeting report

### Advanced

**For maximum control:**
- Automatic hreflang generation with Hugo
- Custom x-default logic
- Language detection and redirect
- Regional content variations
- Hreflang for dynamic content

## Benefits

International SEO. Correct language served to users globally.

User Experience. Users see content in their language.

No Duplicate Content. Each language treated as unique.

Regional Targeting. Serve regional variations (en-US vs en-GB).

Better Rankings. Improved visibility in international search results.

## Related

- [canonical-urls.md](./canonical-urls.md) - Canonical and hreflang together
- [sitemaps-robots-txt.md](./sitemaps-robots-txt.md) - Include hreflang in sitemaps
- [meta-descriptions-titles.md](./meta-descriptions-titles.md) - Localized meta tags
- [../../01-hugo-basics/hugo-multilingual.md](../01-hugo-basics/hugo-multilingual.md) - Hugo multilingual setup
- [../../01-hugo-basics/hugo-content-management.md](../01-hugo-basics/hugo-content-management.md) - Content organization
