# Hugo Taxonomies & Sections

Content classification with tags, categories, custom taxonomies, sections, and related content.

## Why It Matters
- Taxonomies enable browsing by topic, author, or any custom dimension
- Hugo auto-generates list pages for every taxonomy term (zero config)
- Sections define content types and control which templates are used

## Taxonomy Configuration
```toml
# config.toml
[taxonomies]
  tag = "tags"
  category = "categories"
  series = "series"           # Custom taxonomy
  author = "authors"          # Custom taxonomy
```

Front matter usage:
```yaml
tags: ["Hugo", "Tutorial"]
categories: ["Web Development"]
series: ["Hugo Deep Dive"]
authors: ["Jane Doe"]
```
Generated URLs: `/tags/hugo/`, `/authors/jane-doe/`, `/series/hugo-deep-dive/`

## Taxonomy Templates
- `layouts/_default/terms.html` -- lists all terms (e.g., all tags)
- `layouts/_default/taxonomy.html` -- lists pages for one term (e.g., posts tagged "Hugo")
- Section-specific: `layouts/taxonomy/series.html`

```go-html-template
<!-- Sort by count, show top 10 -->
{{ range first 10 .Site.Taxonomies.tags.ByCount }}
  <a href="{{ .Page.Permalink }}">{{ .Page.Title }} ({{ .Count }})</a>
{{ end }}
```

## Sections
Sections are top-level directories under `content/`. Each section gets its own `list.html` template lookup.

Use `cascade` in `_index.md` to set defaults for all child pages:
```yaml
# content/blog/_index.md
cascade:
  type: "blog"
  toc: true
```

## Related Content
```toml
# config.toml
[related]
  [[related.indices]]
    name = "tags"
    weight = 100
  [[related.indices]]
    name = "categories"
    weight = 80
```
```go-html-template
{{ $related := .Site.RegularPages.Related . | first 5 }}
```

## Pitfalls
- Don't use inconsistent casing for taxonomy terms ("Hugo" vs "hugo" creates duplicates)
- Don't forget to define custom taxonomies in config (Hugo only enables tags/categories by default)
- Don't overload posts with too many tags (3-10 is ideal)
- Don't confuse `taxonomy.html` (single term page) with `terms.html` (all terms page)
