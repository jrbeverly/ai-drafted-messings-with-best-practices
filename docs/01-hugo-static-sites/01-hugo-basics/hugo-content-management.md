# Hugo Content Management

Front matter, page bundles, content types, archetypes, and Markdown best practices for Hugo.

## Why It Matters
- Front matter controls metadata, SEO, layout selection, and taxonomy assignment
- Page bundles co-locate content with its resources (images, data) for portability
- Archetypes enforce consistent front matter across content types

## Front Matter Essentials

```yaml
---
title: "Post Title"
date: 2026-02-13T10:00:00Z
draft: false
description: "Meta description for SEO."
tags: ["Hugo", "Tutorial"]
categories: ["Web Development"]
slug: "custom-url-slug"        # Override URL segment
aliases: ["/old-url/"]         # Redirects from old URLs
weight: 10                     # Manual ordering (lower = first)
type: "blog"                   # Override content type / layout lookup
---
```

- Use YAML (`---`) delimiters (most readable; TOML uses `+++`, JSON uses `{}`)
- `publishDate` / `expiryDate` for scheduled content; build with `hugo -F` / `hugo -E`
- `cascade` in `_index.md` applies settings to all child pages in a section

## Page Bundles

| Feature | `_index.md` (Branch) | `index.md` (Leaf) |
|---------|----------------------|--------------------|
| Type | Section/list page | Single page |
| Child pages | Yes | No |
| Template | `list.html` | `single.html` |

- **Leaf bundle:** `content/blog/my-post/index.md` + co-located images/data
- **Branch bundle:** `content/blog/_index.md` with child `.md` files

Access page resources in templates:
```go-html-template
{{ $img := .Resources.GetMatch "hero.jpg" }}
<img src="{{ $img.RelPermalink }}" alt="{{ $img.Title }}">
```

## Content Summaries
- **Auto:** Hugo uses first ~70 words
- **Manual:** Insert `<!--more-->` to split summary from full content
- **Custom:** Set `summary:` in front matter

## Archetypes
```yaml
# archetypes/blog.md
---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: true
tags: []
---
```
Create content: `hugo new blog/my-post.md` (matches `archetypes/blog.md`)

## Render Hooks
Override default Markdown rendering by placing templates in `layouts/_default/_markup/`:
- `render-image.html` -- add lazy loading, `<figure>` wrappers
- `render-link.html` -- add `target="_blank"` for external links

## Pitfalls
- Don't mix front matter formats within a project (pick YAML and stick with it)
- Don't store all images in `static/` -- use page bundles for co-location
- Don't skip heading levels in Markdown (h1 then h3)
- Don't nest content deeper than 3 levels
