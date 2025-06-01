# Hugo Shortcodes & Partials

Reusable components: shortcodes for Markdown content, partials for templates.

## Why It Matters
- Shortcodes let content authors embed rich components without writing HTML
- Partials keep templates DRY and enable caching of expensive operations
- Clear separation: shortcodes = content-facing, partials = developer-facing

## Shortcodes
Location: `layouts/shortcodes/`

**Two syntax types:**
- `{{</* name */>}}` -- raw HTML output
- `{{%/* name */%}}` -- processes inner content as Markdown

```go-html-template
<!-- layouts/shortcodes/alert.html -->
<div class="alert alert-{{ .Get 0 }}">{{ .Inner }}</div>
```
Usage: `{{</* alert "warning" */>}}Careful!{{</* /alert */>}}`

**Named params:** `.Get "src"` | **Positional:** `.Get 0`
**Page access:** `.Page.Params.author` | **Markdown inner:** `{{ .Inner | markdownify }}`

**Built-in shortcodes:** `figure`, `youtube`, `gist`, `ref`, `relref`, `tweet`

## Partials
Location: `layouts/partials/`

```go-html-template
<!-- Include with explicit params (preferred) -->
{{ partial "button.html" (dict "url" "/signup" "text" "Sign Up" "style" "primary") }}

<!-- Cached partial (same output on every page) -->
{{ partialCached "footer.html" . }}

<!-- Cached per section -->
{{ partialCached "sidebar.html" . .Section }}

<!-- Partial that returns a value -->
{{ $minutes := partial "reading-time.html" . }}
```

## Decision Matrix

| Use Case | Component |
|----------|-----------|
| Alert/callout in Markdown | Shortcode |
| YouTube embed in content | Shortcode |
| Site header/footer | Partial |
| Navigation menu | Partial |
| Breadcrumbs | Partial |
| Code tabs in content | Shortcode |

## Pitfalls
- Don't assume shortcode parameters exist -- always use `{{ with .Get "param" }}` guards
- Don't use `partialCached` for page-specific content (it returns the cached version)
- Don't pass implicit context (`.`) to partials when explicit `dict` is clearer
- Don't create deeply nested partial call chains
- Don't put business logic in shortcodes or partials
