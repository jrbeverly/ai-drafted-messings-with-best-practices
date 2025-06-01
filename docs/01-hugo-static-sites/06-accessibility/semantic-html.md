# Semantic HTML

Use HTML5 semantic elements to convey document structure and meaning directly in markup, so assistive technologies can build a navigable document outline.

## Why It Matters

- Screen reader users navigate by landmarks and headings -- generic `<div>`s are invisible to these shortcuts
- Semantic elements carry implicit ARIA roles, eliminating most explicit ARIA and reducing mistake surface
- Defining structure in Hugo layout templates ensures every page inherits correct semantics automatically

## Key Recommendations

**Use landmark elements for page regions:**

```html
<header>       <!-- banner (body-level) -->
<nav>          <!-- navigation -->
<main>         <!-- main (one per page) -->
<aside>        <!-- complementary -->
<footer>       <!-- contentinfo (body-level) -->
```

**Maintain strict heading hierarchy:** exactly one `<h1>` per page, never skip levels (`<h1>` then `<h3>`).

**Choose the right container:** `<article>` for self-contained syndicatable content, `<section aria-labelledby="...">` for thematic groupings with a heading, `<div>` only for styling/layout.

**Use `<time datetime="...">` for all dates,** `<dl>` for metadata/key-value pairs, `<ol>` for breadcrumbs and steps, `<figure>`/`<figcaption>` for captioned media.

**Tables must be data-only:** always include `<caption>`, `<th scope="col|row">`, `<thead>`, `<tbody>`. Never use tables for layout.

**Hugo baseof.html pattern:**

```go-html-template
<body>
  <a href="#main-content" class="skip-link">Skip to main content</a>
  <header>{{ partial "site-header.html" . }}</header>
  <main id="main-content">{{ block "main" . }}{{ end }}</main>
  <footer>{{ partial "site-footer.html" . }}</footer>
</body>
```

**Hugo render hooks** add semantics Markdown cannot: `<figure>`/`<figcaption>` for titled images, anchor links on headings, `rel="noopener"` and `<span class="sr-only">(opens in new tab)</span>` on external links.

**Label duplicate landmarks:** multiple `<nav>` elements need `aria-label="Primary"` / `aria-label="Footer"`.

## Pitfalls to Avoid

- Using `<div>` or `<span>` when a semantic element exists
- Skipping heading levels or using more than one `<h1>`
- `<section>` without a heading (no landmark role without an accessible name)
- Tables for layout instead of CSS Grid/Flexbox
- Omitting `<caption>` from data tables
- Adding redundant ARIA roles to semantic elements (`<nav role="navigation">`)
- Nesting `<main>` inside `<article>` or `<section>`
