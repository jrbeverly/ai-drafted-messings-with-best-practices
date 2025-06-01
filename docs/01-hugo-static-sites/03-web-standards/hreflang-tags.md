# Hreflang Tags

Multi-language SEO. International targeting. Language and regional variations. Prevent duplicate content across languages.

## Principle

Tell search engines which language and regional variations of a page exist. Serve the correct language version to users. Consolidate SEO signals across translations.

## What is Hreflang?

**Hreflang:** HTML attribute that specifies the language and geographic targeting of a page

**Purpose:** Help search engines:
- Show the correct language version in search results
- Understand relationship between translated pages
- Avoid duplicate content penalties for translations

**Format:** `<link rel="alternate" hreflang="...">`

**Specification:** https://developers.google.com/search/docs/specialty/international/localized-versions

## Basic Hreflang Tag

```html
<!-- English version -->
<link rel="alternate" hreflang="en" href="https://example.com/page">

<!-- Spanish version -->
<link rel="alternate" hreflang="es" href="https://example.com/es/page">

<!-- French version -->
<link rel="alternate" hreflang="fr" href="https://example.com/fr/page">
```

**Location:** In the `<head>` section of HTML

**All pages must cross-reference each other** (bidirectional linking).

## Hreflang Format

### Language Only

```html
<link rel="alternate" hreflang="en" href="https://example.com/page">
<link rel="alternate" hreflang="es" href="https://example.com/es/page">
<link rel="alternate" hreflang="de" href="https://example.com/de/page">
```

**Format:** ISO 639-1 language code (2 letters)

**Use when:** Content is for a language regardless of region

### Language + Region

```html
<link rel="alternate" hreflang="en-US" href="https://example.com/us/page">
<link rel="alternate" hreflang="en-GB" href="https://example.com/uk/page">
<link rel="alternate" hreflang="en-AU" href="https://example.com/au/page">
```

**Format:** `language-REGION` (ISO 639-1 + ISO 3166-1 Alpha 2)

**Use when:** Content is tailored for a specific region (e.g., pricing, spelling)

### X-Default (Fallback)

```html
<link rel="alternate" hreflang="x-default" href="https://example.com/page">
<link rel="alternate" hreflang="en" href="https://example.com/page">
<link rel="alternate" hreflang="es" href="https://example.com/es/page">
```

**Purpose:** Default page when no language matches user's preference

**Typically:** Homepage or language selector page

## Complete Hreflang Implementation

**Each page must include:**
1. Hreflang tag for itself (self-referencing)
2. Hreflang tags for all language/region variants
3. X-default fallback

**Example (on English page):**

```html
<!-- On: https://example.com/page -->

<!-- Self-referencing -->
<link rel="alternate" hreflang="en" href="https://example.com/page">

<!-- Other language versions -->
<link rel="alternate" hreflang="es" href="https://example.com/es/page">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/page">
<link rel="alternate" hreflang="de" href="https://example.com/de/page">

<!-- Default fallback -->
<link rel="alternate" hreflang="x-default" href="https://example.com/page">
```

**Same tags on Spanish page:**

```html
<!-- On: https://example.com/es/page -->

<!-- Exactly the same hreflang tags as English page -->
<link rel="alternate" hreflang="en" href="https://example.com/page">
<link rel="alternate" hreflang="es" href="https://example.com/es/page">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/page">
<link rel="alternate" hreflang="de" href="https://example.com/de/page">
<link rel="alternate" hreflang="x-default" href="https://example.com/page">
```

**Bidirectional:** All pages reference each other with identical hreflang tags.

## Hugo Implementation

### Hugo Multilingual Configuration

```toml
# config.toml
defaultContentLanguage = "en"

[languages]
  [languages.en]
    languageCode = "en-US"
    languageName = "English"
    weight = 1
    contentDir = "content/en"

  [languages.es]
    languageCode = "es-ES"
    languageName = "Español"
    weight = 2
    contentDir = "content/es"

  [languages.fr]
    languageCode = "fr-FR"
    languageName = "Français"
    weight = 3
    contentDir = "content/fr"
```

### Hugo Hreflang Template

```go-html-template
{{/* layouts/partials/hreflang.html */}}

{{- if .IsTranslated }}
  {{- range .Translations }}
<link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">
  {{- end }}
{{- end }}

<!-- Self-referencing -->
<link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">

<!-- X-default fallback (usually the default language homepage) -->
<link rel="alternate" hreflang="x-default" href="{{ .Site.Home.Permalink }}">
```

### Advanced Hugo Template

```go-html-template
{{/* layouts/partials/hreflang-advanced.html */}}

{{- if .IsTranslated }}
  {{/* All translations */}}
  {{- range .AllTranslations }}
<link rel="alternate" hreflang="{{ .Language.LanguageCode }}" href="{{ .Permalink }}">
  {{- end }}

  {{/* X-default - link to the default language version */}}
  {{- $defaultLang := .Site.Language.Lang }}
  {{- with .GetPage "/" }}
<link rel="alternate" hreflang="x-default" href="{{ .Permalink }}">
  {{- end }}
{{- else }}
  {{/* Page not translated - just self-reference */}}
<link rel="alternate" hreflang="{{ .Language.LanguageCode }}" href="{{ .Permalink }}">
{{- end }}
```

### Regional Variants (en-US, en-GB)

```toml
# config.toml
[languages]
  [languages.en-us]
    languageCode = "en-US"
    languageName = "English (US)"
    weight = 1
    contentDir = "content/en-us"

  [languages.en-gb]
    languageCode = "en-GB"
    languageName = "English (UK)"
    weight = 2
    contentDir = "content/en-gb"

  [languages.en-au]
    languageCode = "en-AU"
    languageName = "English (AU)"
    weight = 3
    contentDir = "content/en-au"
```

```go-html-template
{{/* Hreflang will use languageCode (en-US, en-GB, etc.) */}}
{{- range .AllTranslations }}
<link rel="alternate" hreflang="{{ .Language.LanguageCode }}" href="{{ .Permalink }}">
{{- end }}
```

## Common Patterns

### Simple Multi-Language Site

**Structure:**
```
example.com/          (English - default)
example.com/es/       (Spanish)
example.com/fr/       (French)
```

**Hreflang:**
```html
<link rel="alternate" hreflang="en" href="https://example.com/page">
<link rel="alternate" hreflang="es" href="https://example.com/es/page">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/page">
<link rel="alternate" hreflang="x-default" href="https://example.com/page">
```

### Separate Domains per Language

**Structure:**
```
example.com     (English)
example.es      (Spanish)
example.fr      (French)
```

**Hreflang:**
```html
<link rel="alternate" hreflang="en" href="https://example.com/page">
<link rel="alternate" hreflang="es" href="https://example.es/page">
<link rel="alternate" hreflang="fr" href="https://example.fr/page">
<link rel="alternate" hreflang="x-default" href="https://example.com/page">
```

### Subdomains per Language

**Structure:**
```
www.example.com     (English)
es.example.com      (Spanish)
fr.example.com      (French)
```

**Hreflang:**
```html
<link rel="alternate" hreflang="en" href="https://www.example.com/page">
<link rel="alternate" hreflang="es" href="https://es.example.com/page">
<link rel="alternate" hreflang="fr" href="https://fr.example.com/page">
<link rel="alternate" hreflang="x-default" href="https://www.example.com/page">
```

### Regional Variants (e.g., Spanish for Spain vs Latin America)

```html
<link rel="alternate" hreflang="es-ES" href="https://example.com/es/page">
<link rel="alternate" hreflang="es-MX" href="https://example.com/mx/page">
<link rel="alternate" hreflang="es-AR" href="https://example.com/ar/page">
<link rel="alternate" hreflang="x-default" href="https://example.com/es/page">
```

## Hreflang in Sitemap

**Alternative to HTML tags:** Specify hreflang in sitemap.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
        xmlns:xhtml="http://www.w3.org/1999/xhtml">
  <url>
    <loc>https://example.com/page</loc>

    <!-- Hreflang in sitemap -->
    <xhtml:link rel="alternate" hreflang="en" href="https://example.com/page"/>
    <xhtml:link rel="alternate" hreflang="es" href="https://example.com/es/page"/>
    <xhtml:link rel="alternate" hreflang="fr" href="https://example.com/fr/page"/>
    <xhtml:link rel="alternate" hreflang="x-default" href="https://example.com/page"/>
  </url>

  <url>
    <loc>https://example.com/es/page</loc>

    <!-- Same hreflang tags -->
    <xhtml:link rel="alternate" hreflang="en" href="https://example.com/page"/>
    <xhtml:link rel="alternate" hreflang="es" href="https://example.com/es/page"/>
    <xhtml:link rel="alternate" hreflang="fr" href="https://example.com/fr/page"/>
    <xhtml:link rel="alternate" hreflang="x-default" href="https://example.com/page"/>
  </url>
</urlset>
```

**Hugo sitemap with hreflang:**

```go-html-template
{{/* layouts/_default/sitemap.xml */}}
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
        xmlns:xhtml="http://www.w3.org/1999/xhtml">
  {{- range .Data.Pages }}
  <url>
    <loc>{{ .Permalink }}</loc>

    {{- if .IsTranslated }}
      {{- range .AllTranslations }}
    <xhtml:link rel="alternate" hreflang="{{ .Language.LanguageCode }}" href="{{ .Permalink }}"/>
      {{- end }}
    {{- end }}
  </url>
  {{- end }}
</urlset>
```

## Testing Hreflang

### Google Search Console

**International Targeting Report:**
1. Go to Search Console
2. Navigate to "Legacy tools and reports" → "International Targeting"
3. Check "Language" tab
4. Review hreflang errors

**URL Inspection Tool:**
1. Inspect a URL
2. Check "Alternate pages" section
3. Verify hreflang tags are detected

### Hreflang Validation Tools

**Aleyda Solis Hreflang Validator:**
https://www.aleydasolis.com/english/international-seo-tools/hreflang-tags-generator/

**Merkle Hreflang Tag Testing Tool:**
https://technicalseo.com/tools/hreflang/

**Manual Check:**

```bash
# Extract hreflang tags from page
curl https://example.com/page | grep "hreflang"

# Check all languages reference each other
curl https://example.com/page | grep 'hreflang="es"'
curl https://example.com/es/page | grep 'hreflang="en"'
```

## Common Mistakes

❌ **Non-bidirectional links:**
```html
<!-- On EN page: -->
<link rel="alternate" hreflang="es" href="https://example.com/es/page">

<!-- On ES page: Missing hreflang back to EN -->
<!-- ❌ This is wrong! Must be bidirectional -->
```

❌ **Missing self-referencing hreflang:**
```html
<!-- On EN page: -->
<!-- Missing self-reference -->
<link rel="alternate" hreflang="es" href="https://example.com/es/page">

<!-- Correct - include self -->
<link rel="alternate" hreflang="en" href="https://example.com/page">
<link rel="alternate" hreflang="es" href="https://example.com/es/page">
```

❌ **Relative URLs:**
```html
<!-- Wrong -->
<link rel="alternate" hreflang="es" href="/es/page">

<!-- Correct -->
<link rel="alternate" hreflang="es" href="https://example.com/es/page">
```

❌ **Invalid language codes:**
```html
<!-- Wrong -->
<link rel="alternate" hreflang="english" href="https://example.com/page">

<!-- Correct (ISO 639-1) -->
<link rel="alternate" hreflang="en" href="https://example.com/page">
```

❌ **Pointing to different content:**
```html
<!-- Wrong - hreflang should point to equivalent content in another language -->
<!-- EN page about dogs -->
<link rel="alternate" hreflang="es" href="https://example.com/es/cats">

<!-- Correct -->
<link rel="alternate" hreflang="es" href="https://example.com/es/dogs">
```

❌ **Using hreflang with canonical pointing elsewhere:**
```html
<!-- Wrong - canonical and hreflang conflict -->
<link rel="canonical" href="https://example.com/other-page">
<link rel="alternate" hreflang="es" href="https://example.com/es/page">

<!-- Correct - canonical self-references, hreflang points to translations -->
<link rel="canonical" href="https://example.com/page">
<link rel="alternate" hreflang="en" href="https://example.com/page">
<link rel="alternate" hreflang="es" href="https://example.com/es/page">
```

## Hreflang + Canonical

**Both should be present:**

```html
<!-- Canonical points to self (preferred version) -->
<link rel="canonical" href="https://example.com/page">

<!-- Hreflang points to language variations -->
<link rel="alternate" hreflang="en" href="https://example.com/page">
<link rel="alternate" hreflang="es" href="https://example.com/es/page">
```

**Canonical:** Specifies preferred URL for THIS language
**Hreflang:** Specifies alternative languages

## Language Codes Reference

**Common Language Codes (ISO 639-1):**

| Code | Language |
|------|----------|
| en | English |
| es | Spanish |
| fr | French |
| de | German |
| it | Italian |
| pt | Portuguese |
| zh | Chinese |
| ja | Japanese |
| ko | Korean |
| ar | Arabic |
| ru | Russian |
| nl | Dutch |

**Common Regional Variants:**

| Code | Description |
|------|-------------|
| en-US | English (United States) |
| en-GB | English (United Kingdom) |
| en-AU | English (Australia) |
| en-CA | English (Canada) |
| es-ES | Spanish (Spain) |
| es-MX | Spanish (Mexico) |
| es-AR | Spanish (Argentina) |
| pt-BR | Portuguese (Brazil) |
| pt-PT | Portuguese (Portugal) |
| zh-CN | Chinese (Simplified, China) |
| zh-TW | Chinese (Traditional, Taiwan) |
| fr-FR | French (France) |
| fr-CA | French (Canada) |

## Best Practices

**Required:**
- Bidirectional hreflang (all pages reference each other)
- Self-referencing hreflang on each page
- Absolute URLs (https://)
- Valid ISO language/region codes

**Recommended:**
- Include x-default for fallback
- Use sitemap.xml for large sites (easier maintenance)
- Test in Google Search Console
- Monitor International Targeting Report

**Optional:**
- Regional variants (en-US vs en-GB) when content differs
- Separate domains/subdomains per language

## Hugo Complete Example

```go-html-template
{{/* layouts/partials/hreflang.html */}}

{{- if .Site.IsMultiLingual }}
  {{- $currentLang := .Language.LanguageCode }}

  {{/* All language versions of this page */}}
  {{- range .AllTranslations }}
<link rel="alternate" hreflang="{{ .Language.LanguageCode }}" href="{{ .Permalink }}">
  {{- end }}

  {{/* X-default fallback (default language homepage or this page in default lang) */}}
  {{- with .Site.GetPage "/" }}
    {{- $defaultPage := . }}
    {{- if $.Translations }}
      {{- range $.AllTranslations }}
        {{- if eq .Language.Lang $.Site.DefaultContentLanguage }}
          {{- $defaultPage = . }}
        {{- end }}
      {{- end }}
    {{- end }}
<link rel="alternate" hreflang="x-default" href="{{ $defaultPage.Permalink }}">
  {{- end }}
{{- else }}
  {{/* Single language site - self-referencing only */}}
<link rel="alternate" hreflang="{{ .Site.Language.LanguageCode }}" href="{{ .Permalink }}">
{{- end }}
```

## Guidelines

**Essential:**
- Bidirectional linking (all pages reference all versions)
- Self-referencing hreflang
- Absolute URLs
- Valid language codes (ISO 639-1)

**Recommended:**
- Include x-default
- Use sitemap.xml for 100+ pages
- Test in Search Console
- Monitor errors regularly

**Best Practices:**
- One hreflang implementation method (HTML OR sitemap, not both)
- Match content across translations
- Use regional codes only when content differs
- Keep URLs consistent across languages

## Benefits

Targeted. Right language shown to right users.

SEO. Consolidates signals across translations.

International. Proper multi-language/region support.

Discoverable. Search engines understand language structure.

## Related

- [canonical-urls.md](./canonical-urls.md) - Canonical tags
- [meta-tags-seo.md](./meta-tags-seo.md) - HTML meta tags
- [sitemap-xml-advanced.md](./sitemap-xml-advanced.md) - XML sitemaps
- [open-graph-protocol.md](./open-graph-protocol.md) - og:locale property
