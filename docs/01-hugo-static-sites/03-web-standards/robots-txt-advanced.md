# robots.txt Advanced

Crawl directives controlling how search engines and bots interact with your site.

## Why It Matters

- Controls crawl behavior and server load from bots
- Directs crawlers to sitemaps for better indexing
- Blocks AI scrapers and aggressive crawlers from consuming content

## Basic Template

```text
User-agent: *
Allow: /
Sitemap: https://example.com/sitemap.xml
```

## Hugo Dynamic Generation

```go-html-template
{{/* layouts/robots.txt */}}
User-agent: *
{{ if hugo.IsProduction }}Allow: /{{ else }}Disallow: /{{ end }}
Sitemap: {{ .Site.BaseURL }}sitemap.xml
```

```yaml
# config.yaml
enableRobotsTXT: true
```

## Block AI Scrapers

```text
User-agent: GPTBot
Disallow: /

User-agent: ChatGPT-User
Disallow: /

User-agent: Google-Extended
Disallow: /

User-agent: anthropic-ai
Disallow: /

User-agent: CCBot
Disallow: /

User-agent: PerplexityBot
Disallow: /
```

## Block Aggressive Crawlers

```text
User-agent: AhrefsBot
Disallow: /

User-agent: SemrushBot
Disallow: /

User-agent: MJ12bot
Disallow: /
```

## Common Disallow Patterns

```text
Disallow: /admin/
Disallow: /api/
Disallow: /search
Disallow: /*?*           # Query string pages
Disallow: /*?utm_source= # Tracking parameters
```

## Key Rules

- Sitemap URLs must be **absolute** (`https://...`)
- `Crawl-delay` is honored by Bing/Yandex but **ignored by Google**
- More specific `User-agent` blocks override the wildcard `*` block
- Do not block `/css/` or `/js/` (Google needs them for rendering)

## Pitfalls to Avoid

- `Disallow: /` on production (blocks entire site)
- Relative sitemap URLs (`/sitemap.xml` instead of full URL)
- Using robots.txt as a **security** mechanism (bots can ignore it)
- Blocking CSS/JS files (hurts SEO rendering)
- Forgetting staging environments still get indexed without `Disallow: /`

## Related

- [well-known-directory.md](./well-known-directory.md)
