# ARIA Labels and Roles

Use ARIA attributes to communicate meaning, structure, and state to assistive technologies -- but only when native HTML semantics are insufficient.

## Why It Matters

- Screen reader users navigate by landmarks, headings, and ARIA states -- missing attributes mean invisible content
- Interactive widgets (tabs, accordions, menus) have no native HTML equivalent and require ARIA roles
- Decorative content without `aria-hidden` creates noise in the accessibility tree

## Key Recommendations

**First rule: prefer native HTML over ARIA.**

```html
<!-- DO: native element (implicit role) -->
<nav>...</nav>
<!-- DON'T: ARIA on generic element -->
<div role="navigation">...</div>
```

**Label duplicate landmarks.** Multiple `<nav>` elements need distinct names:

```html
<nav aria-label="Primary">...</nav>
<nav aria-label="Footer">...</nav>
```

**Use `aria-current="page"` on active nav links (Hugo):**

```go-html-template
<a href="{{ .URL }}"
  {{ if eq $.RelPermalink .URL }}aria-current="page"{{ end }}>
  {{ .Name }}
</a>
```

**Toggle `aria-expanded` on disclosure widgets:**

```html
<button aria-expanded="false" aria-controls="panel-1">Section</button>
<div id="panel-1" hidden>Content...</div>
```

**Hide decorative SVGs:**

```html
<button aria-label="Search">
  <svg aria-hidden="true" focusable="false"><use href="#icon-search"></use></svg>
</button>
```

**Live regions for dynamic updates:**

```html
<div role="status" aria-live="polite" aria-atomic="true">
  12 results found
</div>
```

**Labeling comparison:** `aria-label` = no visible label exists. `aria-labelledby` = visible label elsewhere. `aria-describedby` = supplementary help text or errors.

## Pitfalls to Avoid

- Adding redundant roles to semantic elements (`<nav role="navigation">`)
- Using `aria-label` on non-interactive generic elements (`<div>`, `<span>`) -- ignored by most screen readers
- Placing focusable elements inside `aria-hidden="true"` containers
- Using `aria-live="assertive"` for non-urgent messages
- Referencing nonexistent IDs in `aria-labelledby` or `aria-describedby`
- Forgetting to update `aria-expanded` when toggling content visibility
- Using ARIA as a substitute for proper semantic HTML structure
