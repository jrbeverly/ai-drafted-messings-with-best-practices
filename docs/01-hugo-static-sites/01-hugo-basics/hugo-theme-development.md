# Hugo Theme Development

Template hierarchy, Go templates, base templates, partials, and building custom Hugo themes.

## Why It Matters
- Themes separate presentation from content, enabling reuse across sites
- Understanding the template lookup order prevents unexpected rendering
- Go template syntax is the foundation of all Hugo layout logic

## Template Lookup Order
For a blog post at `content/blog/my-post.md`:
1. `layouts/blog/single.html` (project, section-specific)
2. `layouts/_default/single.html` (project, default)
3. `themes/{name}/layouts/blog/single.html` (theme, section-specific)
4. `themes/{name}/layouts/_default/single.html` (theme, default)

Project files always override theme files at the same path.

## Core Templates

**baseof.html** -- site-wide HTML skeleton with named blocks:
```go-html-template
{{ block "main" . }}{{ end }}     <!-- Overridden by child templates -->
{{ partial "header.html" . }}     <!-- Reusable partial -->
```

**single.html / list.html** -- override the `"main"` block:
```go-html-template
{{ define "main" }}
  <h1>{{ .Title }}</h1>
  {{ .Content }}
{{ end }}
```

## Go Template Quick Reference
```go-html-template
{{ .Title }}                      <!-- Page variable -->
{{ .Site.Params.author }}         <!-- Site config param -->
{{ with .Params.author }}...{{ end }}  <!-- Nil-safe check -->
{{ range .Pages }}...{{ end }}    <!-- Loop -->
{{ if and .IsPage (not .Draft) }} <!-- Conditional -->
{{ .Date.Format "January 2, 2006" }}  <!-- Date formatting -->
{{ $var := .Title | lower | title }}   <!-- Pipes + variables -->
```

## Partials
- Location: `layouts/partials/`
- Pass context explicitly: `{{ partial "card.html" (dict "title" .Title "url" .Permalink) }}`
- Use `partialCached` for expensive, site-wide partials (footer, header)
- Partials can return values with `{{ return $value }}`

## Creating a Theme
```bash
hugo new theme mytheme            # Generates skeleton
```
Key files: `baseof.html`, `single.html`, `list.html`, `head.html`, `header.html`, `footer.html`

Include an `exampleSite/` directory and `theme.toml` with metadata.

## Quick Reference
```toml
# config.toml
theme = "mytheme"
paginate = 10
```

## Pitfalls
- Don't forget to pass context (`.`) to partials and blocks
- Don't hard-code values in themes -- use `.Site.Params`
- Don't duplicate HTML across templates -- extract to partials
- Don't nest template logic deeper than 3 levels
- Don't put business logic in templates
