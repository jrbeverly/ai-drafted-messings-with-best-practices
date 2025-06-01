# Screen Reader Optimization

Optimize Hugo templates and content for how assistive technology actually parses, navigates, and announces web pages.

## Why It Matters

- Screen readers build an accessibility tree from the DOM -- not the visual layout -- so semantic structure determines navigability
- Users navigate by headings (H key), landmarks (D key), and links (K key); missing semantics mean invisible content
- Content authored across Hugo layouts, render hooks, and Markdown must produce a logical, noise-free accessibility tree

## Key Recommendations

**Include `.sr-only` CSS in every Hugo project** for visually hidden screen-reader-only text:

```css
.sr-only {
  position: absolute; width: 1px; height: 1px;
  padding: 0; margin: -1px; overflow: hidden;
  clip: rect(0,0,0,0); white-space: nowrap; border: 0;
}
```

**Alt text rules:** describe what the image communicates (not what it looks like), keep to 10-150 chars, never start with "Image of", use `alt=""` for decorative images. Hugo image render hook should enforce alt text presence.

**Link text must make sense out of context** (screen readers list all links). Avoid "click here" and "read more" without appended `<span class="sr-only"> about [topic]</span>`.

**Set `lang` on `<html>`** via Hugo's `site.Language.Lang`. Wrap inline foreign text in `<span lang="fr">` for correct pronunciation.

**DOM order = reading order.** Never rely on CSS `order`, `flex-direction: row-reverse`, or absolute positioning to create reading sequence.

**Live regions must exist in DOM at page load** (not created dynamically). Use `aria-live="polite"` for search results and status updates, `"assertive"` only for critical errors.

**Tables need `<caption>`, `<th scope="col|row">`** for cell-by-cell navigation with header announcements.

**External links:** Hugo render hook should add `rel="noopener"`, `target="_blank"`, and `<span class="sr-only">(opens in new tab)</span>`.

**Technique comparison:** `.sr-only` = adds text alongside visible content. `aria-label` = replaces accessible name entirely. `aria-hidden="true"` = removes from accessibility tree. Never use both `.sr-only` and `aria-label` on the same element.

## Pitfalls to Avoid

- Omitting the `alt` attribute entirely (worse than bad alt text)
- Starting alt text with "Image of" or "Photo of"
- Same generic link text ("Read more") for multiple destinations
- Wrapping entire block content in a single `<a>` element
- Creating live regions dynamically after page load
- Using `aria-live="assertive"` for non-urgent updates
- Using tables for page layout
- Skipping screen reader testing because automated tools passed
