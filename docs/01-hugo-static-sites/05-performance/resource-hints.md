# Resource Hints Strategy & Decision Framework

Treat resource hints as a performance budget, not a wish list. Each hint consumes browser resources and competes with critical work.

## Why It Matters

- Misapplied hints hurt performance (too many preloads delay critical resources)
- Strategic hints reduce LCP by 200-500ms and make navigation feel instant
- Centralized management in one Hugo partial prevents inconsistency

## Decision Tree

```
Resource needed on CURRENT page?
  YES -> Third-party origin?
    YES, critical  -> preconnect (+ dns-prefetch fallback)
    YES, not critical -> dns-prefetch only
    NO, above-fold critical -> preload
    NO, not critical -> no hint needed
  NO -> Confidence user navigates there?
    >80% -> Speculation Rules (prerender, moderate)
    50-80% -> prefetch
    <50% -> no hint
```

## Resource Budget

| Hint Type | Max per Page | Risk If Exceeded |
|---|---|---|
| `preload` | 2-3 | Competes with critical resources |
| `preconnect` | 2-4 origins | Exhausts connection pool |
| `dns-prefetch` | 6-8 origins | Minimal risk |
| `prefetch` | 2-4 resources | Wastes data on metered connections |
| Speculation Rules (prerender) | 1-2 pages | Expensive CPU/memory/bandwidth |

Total: no more than ~10-15 hints across all types.

## Correct Order in `<head>`

1. `preconnect` + `dns-prefetch`
2. `preload`
3. Stylesheets
4. `prefetch`
5. Scripts
6. Speculation Rules

## Hugo Centralized Partial

Manage all hints from `layouts/partials/head/resource-hints.html`. Configure origins in `hugo.toml`:

```toml
[[params.preconnect]]
  url = "https://api.example.com"
  crossorigin = true
dnsPrefetch = ["https://analytics.example.com"]
preloadFonts = ["fonts/inter-regular.woff2"]
```

## Page-Type Aware Hints

Different pages need different hints. Homepage preloads hero image; blog post prefetches next article; landing page prefetches CTA target.

## Speculation Rules

```html
<script type="speculationrules">
{ "prerender": [{ "where": {
    "and": [{ "href_matches": "/*" },
            { "not": { "href_matches": "/admin/*" } },
            { "not": { "selector_matches": "[data-no-prerender]" } }]
  }, "eagerness": "conservative" }] }
</script>
```

Chromium-only; other browsers safely ignore it. Start with `conservative` or `moderate`.

## Measuring Effectiveness

- Chrome DevTools Network tab: enable Priority column, check waterfall timing
- Lighthouse: "Preload key requests", "Preconnect to required origins" audits
- Compare LCP/FCP before and after adding hints

## Key Recommendations

- Use the decision tree to select the correct hint type
- Stay within budget (2-3 preloads, 2-4 preconnects per page)
- Centralize hints in a single Hugo partial
- Vary hints by page type
- Measure before and after with Lighthouse

## Pitfalls

- Adding hints without measuring impact
- Preloading everything (cancels priority benefit)
- Preconnecting to origins not used on current page
- Speculation Rules without excluding admin/logout/API pages
- Hardcoding fingerprinted URLs instead of using Hugo variables
