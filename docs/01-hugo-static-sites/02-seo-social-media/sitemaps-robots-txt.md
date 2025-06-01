# Sitemaps and robots.txt

XML sitemaps. robots.txt configuration. Search engine crawling. Sitemap submission. Hugo sitemap template.

## Principle

Generate XML sitemaps to help search engines discover and index your content. Use robots.txt to control crawler access. Submit sitemaps to search engines for faster indexing. Optimize crawl budget for better SEO.

## XML Sitemaps

**What is a sitemap?**
- XML file listing all pages on your site
- Helps search engines discover content
- Includes metadata (last modified, priority, change frequency)

**Sitemap protocol:** https://www.sitemaps.org/

**Location:** https://example.com/sitemap.xml

**Why use sitemaps:**
- Faster indexing of new content
- Better discovery of deep pages
- Communicate page importance
- Track indexing in Google Search Console

## Hugo Built-in Sitemap

Hugo automatically generates `sitemap.xml` for your entire site.

**Default output:** `public/sitemap.xml`

**URL:** https://yoursite.com/sitemap.xml

### Default Sitemap Template

Hugo generates sitemaps automatically with no configuration required.

**Example output:**

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
  xmlns:xhtml="http://www.w3.org/1999/xhtml">
  <url>
    <loc>https://example.com/</loc>
    <lastmod>2026-02-13T10:00:00+00:00</lastmod>
  </url>
  <url>
    <loc>https://example.com/blog/post-1/</loc>
    <lastmod>2026-02-13T10:00:00+00:00</lastmod>
  </url>
  <url>
    <loc>https://example.com/blog/post-2/</loc>
    <lastmod>2026-02-12T15:30:00+00:00</lastmod>
  </url>
</urlset>
```

### Sitemap Configuration

**config.toml:**

```toml
baseURL = "https://example.com"

[sitemap]
  changefreq = "monthly"
  filename = "sitemap.xml"
  priority = 0.5
```

**Parameters:**
- `changefreq` - How often page changes (always, hourly, daily, weekly, monthly, yearly, never)
- `filename` - Sitemap filename (default: sitemap.xml)
- `priority` - Page priority 0.0-1.0 (default: 0.5)

**Note:** `priority` and `changefreq` are hints, not directives. Search engines may ignore them.

### Per-Page Sitemap Control

**Front matter:**

```yaml
---
title: "My Blog Post"
date: 2026-02-13
sitemap:
  changefreq: weekly
  priority: 0.8
---
```

**Exclude page from sitemap:**

```yaml
---
title: "Draft Page"
sitemap:
  disable: true
---
```

**Or:**

```yaml
---
title: "Private Page"
_build:
  list: false  # Excludes from sitemap
---
```

### Custom Sitemap Template

**Override default template:** `layouts/sitemap.xml`

```xml
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
  xmlns:xhtml="http://www.w3.org/1999/xhtml">
  {{ range .Data.Pages }}
  {{ if not .Params.sitemap.disable }}
  <url>
    <loc>{{ .Permalink }}</loc>
    {{ if not .Lastmod.IsZero }}
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05-07:00" | safeHTML }}</lastmod>
    {{ end }}
    {{ with .Sitemap.ChangeFreq }}
    <changefreq>{{ . }}</changefreq>
    {{ end }}
    {{ with .Sitemap.Priority }}
    <priority>{{ . }}</priority>
    {{ end }}
  </url>
  {{ end }}
  {{ end }}
</urlset>
```

### Section-Specific Priorities

**Set different priorities by content section:**

```xml
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  {{ range .Data.Pages }}
  {{ $priority := 0.5 }}
  {{ $changefreq := "monthly" }}

  {{ if .IsHome }}
    {{ $priority = 1.0 }}
    {{ $changefreq = "daily" }}
  {{ else if eq .Section "blog" }}
    {{ $priority = 0.8 }}
    {{ $changefreq = "weekly" }}
  {{ else if eq .Section "docs" }}
    {{ $priority = 0.9 }}
    {{ $changefreq = "weekly" }}
  {{ end }}

  {{ with .Params.sitemap.priority }}
    {{ $priority = . }}
  {{ end }}

  {{ with .Params.sitemap.changefreq }}
    {{ $changefreq = . }}
  {{ end }}

  <url>
    <loc>{{ .Permalink }}</loc>
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
    <changefreq>{{ $changefreq }}</changefreq>
    <priority>{{ $priority }}</priority>
  </url>
  {{ end }}
</urlset>
```

### Image Sitemap Extension

**Include images in sitemap:**

```xml
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
  xmlns:image="http://www.google.com/schemas/sitemap-image/1.1">
  {{ range .Data.Pages }}
  <url>
    <loc>{{ .Permalink }}</loc>
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
    {{ range .Params.images }}
    <image:image>
      <image:loc>{{ . | absURL }}</image:loc>
      <image:caption>{{ $.Title }}</image:caption>
    </image:image>
    {{ end }}
  </url>
  {{ end }}
</urlset>
```

### News Sitemap Extension

**For news sites (special Google News sitemap):**

```xml
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
  xmlns:news="http://www.google.com/schemas/sitemap-news/0.9">
  {{ range where .Data.Pages "Section" "news" }}
  {{ if ge .Date.Unix (now.AddDate 0 0 -2).Unix }}
  <url>
    <loc>{{ .Permalink }}</loc>
    <news:news>
      <news:publication>
        <news:name>{{ $.Site.Title }}</news:name>
        <news:language>{{ $.Site.Language.Lang }}</news:language>
      </news:publication>
      <news:publication_date>{{ .Date.Format "2006-01-02T15:04:05-07:00" }}</news:publication_date>
      <news:title>{{ .Title }}</news:title>
    </news:news>
  </url>
  {{ end }}
  {{ end }}
</urlset>
```

**Save as:** `layouts/section/news-sitemap.xml`

**Access at:** https://example.com/news/sitemap.xml

## robots.txt

**What is robots.txt?**
- Text file controlling search engine crawler access
- Specifies which pages to crawl/not crawl
- Links to sitemap

**Location:** https://example.com/robots.txt

**Robots Exclusion Protocol:** https://developers.google.com/search/docs/crawling-indexing/robots/intro

### Hugo Built-in robots.txt

**Enable in config:**

**config.toml:**

```toml
enableRobotsTXT = true
```

**Default output:**

```
User-agent: *
```

### Custom robots.txt Template

**Create:** `layouts/robots.txt`

**Basic example:**

```
User-agent: *
Disallow: /admin/
Disallow: /private/
Allow: /

Sitemap: {{ .Site.BaseURL }}sitemap.xml
```

### Production robots.txt

**layouts/robots.txt:**

```
User-agent: *
Disallow: /admin/
Disallow: /drafts/
Disallow: /search/
Disallow: /*.json$
Allow: /

# Block AI scrapers (optional)
User-agent: GPTBot
Disallow: /

User-agent: CCBot
Disallow: /

User-agent: ChatGPT-User
Disallow: /

User-agent: Google-Extended
Disallow: /

# Crawl-delay for aggressive bots
User-agent: SemrushBot
Crawl-delay: 10

User-agent: AhrefsBot
Crawl-delay: 10

# Sitemap
Sitemap: {{ .Site.BaseURL }}sitemap.xml

# Host (for Yandex)
Host: {{ .Site.BaseURL }}
```

### Environment-Specific robots.txt

**Block all crawlers in staging:**

**layouts/robots.txt:**

```
{{ if eq (getenv "HUGO_ENV") "production" }}
User-agent: *
Allow: /

Sitemap: {{ .Site.BaseURL }}sitemap.xml
{{ else }}
User-agent: *
Disallow: /
{{ end }}
```

**Or use config environments:**

**config/production/config.toml:**

```toml
[params]
  robots_allow = true
```

**config/development/config.toml:**

```toml
[params]
  robots_allow = false
```

**layouts/robots.txt:**

```
{{ if .Site.Params.robots_allow }}
User-agent: *
Allow: /

Sitemap: {{ .Site.BaseURL }}sitemap.xml
{{ else }}
User-agent: *
Disallow: /
{{ end }}
```

### Block Specific Paths

**Block URL parameters:**

```
User-agent: *
Disallow: /*?*  # Block all URLs with query parameters
Allow: /search?*  # Except search
```

**Block file types:**

```
User-agent: *
Disallow: /*.pdf$
Disallow: /*.zip$
Allow: /public/*.pdf  # Except public PDFs
```

**Block sections:**

```
User-agent: *
Disallow: /admin/
Disallow: /dashboard/
Disallow: /api/
Allow: /
```

## Sitemap Submission

### Google Search Console

**URL:** https://search.google.com/search-console

**Steps:**
1. Add and verify your site
2. Go to "Sitemaps" in left sidebar
3. Enter sitemap URL: `sitemap.xml`
4. Click "Submit"

**Monitor:**
- Pages discovered
- Pages indexed
- Errors

### Bing Webmaster Tools

**URL:** https://www.bing.com/webmasters

**Steps:**
1. Add and verify your site
2. Go to "Sitemaps"
3. Submit sitemap URL
4. Monitor indexing status

### Manual Ping

**Google:**

```bash
curl "https://www.google.com/ping?sitemap=https://example.com/sitemap.xml"
```

**Bing:**

```bash
curl "https://www.bing.com/ping?sitemap=https://example.com/sitemap.xml"
```

**After deployment (automated):**

```bash
# In CI/CD pipeline
curl -s "https://www.google.com/ping?sitemap=${SITE_URL}/sitemap.xml"
curl -s "https://www.bing.com/ping?sitemap=${SITE_URL}/sitemap.xml"
```

## Sitemap Best Practices

### Size Limits

**Maximum:**
- 50,000 URLs per sitemap
- 50MB uncompressed

**If exceeded, use sitemap index** (see [sitemap-index.md](./sitemap-index.md))

### URL Requirements

**✅ DO:**
- Use absolute URLs (https://example.com/page/)
- Include trailing slash consistently
- Use HTTPS (if site supports it)
- Include only canonical URLs
- Update `lastmod` when content changes

**❌ DON'T:**
- Include relative URLs (/page/)
- Include redirected URLs
- Include noindex pages
- Include paginated duplicates (page/2/, page/3/)
- Include URL parameters (unless required)

### Priority Values

**Recommended:**
- Homepage: 1.0
- Main sections: 0.8-0.9
- Blog posts: 0.6-0.8
- Archive pages: 0.4-0.6
- Tags/categories: 0.3-0.5

**Note:** Priority is relative within your site, not global.

### Change Frequency

**Realistic values:**
- Homepage: daily or weekly
- Blog: weekly
- Static pages: monthly or yearly
- Documentation: weekly

**Note:** Google ignores this field; use as hint only.

### Last Modified

**Always include `lastmod`:**
- Shows content freshness
- Helps search engines prioritize crawling
- Use actual file modification date

**Hugo automatically uses:**
- Front matter `lastmod`
- Git commit date (if enabled)
- File modification date

**Enable Git dates:**

**config.toml:**

```toml
enableGitInfo = true
```

## robots.txt Best Practices

### Allow vs Disallow

**✅ DO:**
- Allow all by default (`User-agent: * / Allow: /`)
- Explicitly disallow specific paths
- Include sitemap URL
- Use comments for documentation
- Test with Google Search Console

**❌ DON'T:**
- Block CSS/JS files (hurts rendering)
- Block images (hurts image search)
- Use robots.txt as security (it's not!)
- Block entire site in production
- Forget sitemap directive

### User-Agent Targeting

**Common user-agents:**

```
User-agent: Googlebot  # Google
User-agent: Bingbot    # Bing
User-agent: Slurp      # Yahoo
User-agent: DuckDuckBot  # DuckDuckGo
User-agent: Baiduspider  # Baidu
User-agent: YandexBot    # Yandex
```

**Target specific bot:**

```
User-agent: Googlebot
Disallow: /private/

User-agent: *
Disallow: /admin/
Allow: /
```

### Crawl-Delay

**Slow down aggressive crawlers:**

```
User-agent: SemrushBot
Crawl-delay: 10  # 10 seconds between requests

User-agent: AhrefsBot
Crawl-delay: 10
```

**Note:** Google and Bing ignore `Crawl-delay` (use Search Console rate limiting instead).

### AI Scraper Blocking

**Block AI training scrapers (optional):**

```
# OpenAI
User-agent: GPTBot
Disallow: /

User-agent: ChatGPT-User
Disallow: /

# Common Crawl (used by many AI companies)
User-agent: CCBot
Disallow: /

# Google Extended (Bard training)
User-agent: Google-Extended
Disallow: /

# Anthropic
User-agent: Claude-Web
Disallow: /

# Meta
User-agent: FacebookBot
Disallow: /
```

## Testing and Validation

### Google robots.txt Tester

**URL:** https://www.google.com/webmasters/tools/robots-testing-tool

**Or Search Console:**
1. Go to "Settings" > "robots.txt Tester"
2. Test specific URLs
3. Verify allow/disallow rules

### Sitemap Validation

**Online validators:**
- https://www.xml-sitemaps.com/validate-xml-sitemap.html
- Google Search Console (automatic validation)

**Manual check:**

```bash
# Download sitemap
curl https://example.com/sitemap.xml

# Validate XML syntax
curl -s https://example.com/sitemap.xml | xmllint --noout -

# Check URLs are accessible
curl -s https://example.com/sitemap.xml | grep -o '<loc>[^<]*</loc>' | sed 's/<[^>]*>//g' | while read url; do
  echo "Testing $url"
  curl -sI "$url" | head -1
done
```

### Hugo Validation

**Check sitemap generated:**

```bash
hugo
ls -lh public/sitemap.xml
cat public/sitemap.xml
```

**Check robots.txt generated:**

```bash
hugo
cat public/robots.txt
```

**Test locally:**

```bash
hugo server

# Check sitemap
curl http://localhost:1313/sitemap.xml

# Check robots.txt
curl http://localhost:1313/robots.txt
```

## Common Issues

### Sitemap Not Generating

**Problem:** `sitemap.xml` not found

**Solutions:**

1. **Check Hugo generates it:**
   ```bash
   hugo
   ls public/sitemap.xml
   ```

2. **Verify baseURL is set:**
   ```toml
   baseURL = "https://example.com"  # Must be set
   ```

3. **Check pages exist:**
   ```bash
   hugo list all
   ```

4. **Ensure not disabled:**
   ```yaml
   # Don't do this in config:
   disableKinds: ["sitemap"]  # ❌ Disables sitemap
   ```

### robots.txt Not Generating

**Problem:** `robots.txt` not found

**Solution:**

**config.toml:**

```toml
enableRobotsTXT = true  # Must be enabled
```

### Empty Sitemap

**Problem:** Sitemap exists but has no URLs

**Causes:**
- All pages have `sitemap.disable: true`
- All pages have `_build.list: false`
- No published content

**Solution:**

Check front matter:

```yaml
---
title: "My Page"
# Ensure these are not set:
# sitemap:
#   disable: true
# _build:
#   list: false
---
```

### Wrong URLs in Sitemap

**Problem:** URLs are http:// instead of https://

**Solution:**

**config.toml:**

```toml
baseURL = "https://example.com"  # Must use https://
```

### Sitemap Not Updating

**Problem:** Old content in sitemap

**Solution:**

```bash
# Clean and rebuild
hugo --cleanDestinationDir
```

**Check lastmod dates:**

```toml
enableGitInfo = true  # Use Git dates
```

## Complete Examples

### Production config.toml

```toml
baseURL = "https://example.com"
languageCode = "en-us"
title = "Hugo Best Practices"

# Enable robots.txt generation
enableRobotsTXT = true

# Enable Git info for accurate lastmod
enableGitInfo = true

# Sitemap configuration
[sitemap]
  changefreq = "weekly"
  filename = "sitemap.xml"
  priority = 0.5
```

### Production robots.txt

**layouts/robots.txt:**

```
User-agent: *
Allow: /

# Block admin areas
Disallow: /admin/
Disallow: /dashboard/

# Block search results
Disallow: /search/

# Block AI scrapers (optional)
User-agent: GPTBot
Disallow: /

User-agent: CCBot
Disallow: /

# Sitemap
Sitemap: {{ .Site.BaseURL }}sitemap.xml
{{ range .Site.Home.AllTranslations }}
Sitemap: {{ .Permalink }}sitemap.xml
{{ end }}

# Host
Host: {{ .Site.BaseURL }}
```

### Custom Sitemap with Images

**layouts/sitemap.xml:**

```xml
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
  xmlns:image="http://www.google.com/schemas/sitemap-image/1.1"
  xmlns:xhtml="http://www.w3.org/1999/xhtml">
  {{ range .Data.Pages }}
  {{ if not .Params.sitemap.disable }}
  <url>
    <loc>{{ .Permalink }}</loc>
    {{ if not .Lastmod.IsZero }}
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
    {{ end }}
    {{ with .Sitemap.ChangeFreq }}
    <changefreq>{{ . }}</changefreq>
    {{ end }}
    {{ with .Sitemap.Priority }}
    <priority>{{ . }}</priority>
    {{ end }}

    {{/* Images */}}
    {{ range .Params.images }}
    <image:image>
      <image:loc>{{ . | absURL }}</image:loc>
      <image:caption>{{ $.Title }}</image:caption>
    </image:image>
    {{ end }}

    {{/* Alternate language versions */}}
    {{ if .IsTranslated }}
    {{ range .Translations }}
    <xhtml:link rel="alternate" hreflang="{{ .Language.Lang }}" href="{{ .Permalink }}"/>
    {{ end }}
    {{ end }}
  </url>
  {{ end }}
  {{ end }}
</urlset>
```

## Automation

### CI/CD Sitemap Ping

**GitHub Actions (.github/workflows/deploy.yml):**

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'

      - name: Build
        run: hugo --minify

      - name: Deploy
        # Your deployment steps...

      - name: Ping Search Engines
        run: |
          SITE_URL="${{ secrets.SITE_URL }}"
          curl -s "https://www.google.com/ping?sitemap=${SITE_URL}/sitemap.xml"
          curl -s "https://www.bing.com/ping?sitemap=${SITE_URL}/sitemap.xml"
```

### Pre-deployment Validation

```bash
#!/bin/bash
# validate-sitemap.sh

echo "Building site..."
hugo

echo "Validating sitemap..."
if [ ! -f public/sitemap.xml ]; then
  echo "❌ sitemap.xml not found"
  exit 1
fi

echo "Checking XML syntax..."
xmllint --noout public/sitemap.xml
if [ $? -ne 0 ]; then
  echo "❌ Invalid XML"
  exit 1
fi

echo "Checking robots.txt..."
if [ ! -f public/robots.txt ]; then
  echo "❌ robots.txt not found"
  exit 1
fi

echo "✅ All checks passed"
```

## Guidelines

### Essential

**Minimum requirements:**
- Generate sitemap (Hugo default)
- Set correct baseURL
- Enable robots.txt
- Include sitemap URL in robots.txt
- Submit to Google Search Console

**Minimum config.toml:**

```toml
baseURL = "https://example.com"
enableRobotsTXT = true
```

### Recommended

**For better SEO:**
- Custom sitemap template with priorities
- Section-specific change frequencies
- Image sitemap extensions
- Block admin/private sections in robots.txt
- Use Git info for accurate lastmod
- Ping search engines after deployment

### Advanced

**For maximum control:**
- Sitemap index for large sites
- News sitemap for news sites
- Multiple language sitemaps
- Dynamic robots.txt per environment
- Automated testing and validation
- Custom crawl-delay for specific bots

## Performance Checklist

**Before going live:**

- [ ] baseURL is correct (https://example.com)
- [ ] Sitemap generates successfully
- [ ] All pages have lastmod dates
- [ ] Sitemap includes images (if applicable)
- [ ] robots.txt allows main content
- [ ] robots.txt includes sitemap URL
- [ ] No sensitive paths exposed
- [ ] Submitted to Google Search Console
- [ ] Submitted to Bing Webmaster Tools
- [ ] Tested with robots.txt tester
- [ ] Validated sitemap XML syntax
- [ ] Correct environment (staging blocks crawlers)

## Benefits

Faster Indexing. Search engines discover new content quickly.

Better Crawling. Sitemap guides crawlers to important pages.

SEO Improvement. Proper crawling improves search rankings.

Control. robots.txt controls which pages are crawled.

Monitoring. Search Console shows indexing status and errors.

## Related

- [sitemap-index.md](./sitemap-index.md) - Sitemap index for large sites
- [canonical-urls.md](./canonical-urls.md) - Canonical URL specification
- [hreflang-tags.md](./hreflang-tags.md) - Multilingual site markup
- [structured-data-schema-org.md](./structured-data-schema-org.md) - Structured data for search
- [../../01-hugo-basics/hugo-content-management.md](../01-hugo-basics/hugo-content-management.md) - Content organization
