# Hugo Multilingual Sites

Native multi-language support, translation management, i18n strings, and language switching.

## Why It Matters
- Hugo has built-in multilingual support with automatic URL prefixing and translation linking
- Proper hreflang tags and per-language sitemaps are critical for international SEO
- i18n files keep UI strings out of templates, enabling clean translations

## Configuration
```toml
defaultContentLanguage = "en"
defaultContentLanguageInSubdir = false   # /about/ vs /en/about/

[languages.en]
  languageName = "English"
  weight = 1
  contentDir = "content/en"

[languages.es]
  languageName = "Español"
  weight = 2
  contentDir = "content/es"
```

## Content Organization
**Option A: Directory per language** (recommended for clarity)
```
content/en/about.md
content/es/about.md
```

**Option B: Filename suffix**
```
content/about.en.md
content/about.es.md
```

Link translations with different filenames via `translationKey` in front matter.

## i18n Strings
```toml
# i18n/en.toml
[readMore]
other = "Read more"

[minutes]
one = "{{ .Count }} minute"
other = "{{ .Count }} minutes"
```

```go-html-template
{{ i18n "readMore" }}
{{ i18n "minutes" .ReadingTime }}
```

## Language Switcher
```go-html-template
{{ if .IsTranslated }}
  {{ range .Translations }}
    <a href="{{ .Permalink }}" hreflang="{{ .Language.Lang }}">
      {{ .Language.LanguageName }}
    </a>
  {{ end }}
{{ end }}
```

## SEO Essentials
```go-html-template
<html lang="{{ .Language.Lang }}">
<link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">
{{ range .Translations }}
  <link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}">
{{ end }}
```
Hugo auto-generates per-language sitemaps: `/en/sitemap.xml`, `/es/sitemap.xml`

## Pitfalls
- Don't hard-code UI text in templates -- use i18n files
- Don't forget `hreflang` tags (major SEO impact for multilingual sites)
- Don't mix translated and untranslated content in the same directory
- Don't forget language-specific menus: `[[languages.es.menu.main]]`
