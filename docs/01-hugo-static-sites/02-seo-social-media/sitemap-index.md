# Sitemap Index

Sitemap index files. Multiple sitemaps. Large site organization. Section sitemaps. Language-specific sitemaps.

## Principle

Use sitemap index files to organize multiple sitemaps. Split large sitemaps by section, language, or date. Stay under sitemap size limits (50,000 URLs or 50MB). Improve crawl efficiency for large sites.

## What is a Sitemap Index?

**Sitemap Index:** XML file that references multiple sitemap files

**When to use:**
- Site has > 50,000 URLs (sitemap limit)
- Sitemap file > 50MB
- Want to organize sitemaps by section
- Multilingual sites (one sitemap per language)
- Separate sitemaps by content type

**Protocol:** https://www.sitemaps.org/protocol.html#index

**Sitemap limits:**
- Maximum 50,000 URLs per sitemap
- Maximum 50MB uncompressed
- Maximum 50,000 sitemaps per sitemap index

## Sitemap Index Structure

**Basic sitemap index:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap>
    <loc>https://example.com/sitemap-blog.xml</loc>
    <lastmod>2026-02-13T10:00:00+00:00</lastmod>
  </sitemap>
  <sitemap>
    <loc>https://example.com/sitemap-docs.xml</loc>
    <lastmod>2026-02-12T15:30:00+00:00</lastmod>
  </sitemap>
  <sitemap>
    <loc>https://example.com/sitemap-products.xml</loc>
    <lastmod>2026-02-11T09:00:00+00:00</lastmod>
  </sitemap>
</sitemapindex>
```

**Elements:**
- `<sitemapindex>` - Root element
- `<sitemap>` - Individual sitemap reference
- `<loc>` - Absolute URL to sitemap
- `<lastmod>` - When sitemap last updated (optional)

## When to Use Sitemap Index

### Size Limits

**Use sitemap index if:**
- ✅ More than 50,000 total URLs
- ✅ Sitemap file larger than 50MB
- ✅ Expect to exceed limits soon

**Example calculation:**
```
Blog posts: 30,000 URLs
Documentation: 15,000 URLs
Product pages: 10,000 URLs
Total: 55,000 URLs → Use sitemap index
```

### Organization Benefits

**Use sitemap index for better organization:**
- Separate sitemaps by content section (blog, docs, news)
- One sitemap per language (en, es, fr)
- Split by date (2024, 2025, 2026)
- Separate by content type (articles, videos, images)

**Benefits:**
- Easier to debug specific sections
- Better crawl analytics
- Targeted updates (only regenerate changed section)
- Clear structure

## Hugo Implementation

### Hugo Built-in Behavior

Hugo automatically generates sitemap index for multilingual sites.

**For multilingual sites:**

**config.toml:**

```toml
[languages]
  [languages.en]
    languageName = "English"
    weight = 1

  [languages.es]
    languageName = "Español"
    weight = 2

  [languages.fr]
    languageName = "Français"
    weight = 3
```

**Hugo generates:**
- `/sitemap.xml` (sitemap index)
- `/en/sitemap.xml` (English sitemap)
- `/es/sitemap.xml` (Spanish sitemap)
- `/fr/sitemap.xml` (French sitemap)

**Root sitemap index (`/sitemap.xml`):**

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap>
    <loc>https://example.com/en/sitemap.xml</loc>
  </sitemap>
  <sitemap>
    <loc>https://example.com/es/sitemap.xml</loc>
  </sitemap>
  <sitemap>
    <loc>https://example.com/fr/sitemap.xml</loc>
  </sitemap>
</sitemapindex>
```

### Custom Sitemap Index

**For section-based organization:**

**layouts/sitemapindex.xml:**

```xml
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  {{/* Blog sitemap */}}
  <sitemap>
    <loc>{{ .Site.BaseURL }}sitemap-blog.xml</loc>
    {{ with (where .Site.RegularPages "Section" "blog") }}
    {{ with (index . 0) }}
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
    {{ end }}
    {{ end }}
  </sitemap>

  {{/* Documentation sitemap */}}
  <sitemap>
    <loc>{{ .Site.BaseURL }}sitemap-docs.xml</loc>
    {{ with (where .Site.RegularPages "Section" "docs") }}
    {{ with (index . 0) }}
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
    {{ end }}
    {{ end }}
  </sitemap>

  {{/* News sitemap */}}
  <sitemap>
    <loc>{{ .Site.BaseURL }}sitemap-news.xml</loc>
    {{ with (where .Site.RegularPages "Section" "news") }}
    {{ with (index . 0) }}
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
    {{ end }}
    {{ end }}
  </sitemap>

  {{/* Products sitemap */}}
  <sitemap>
    <loc>{{ .Site.BaseURL }}sitemap-products.xml</loc>
    {{ with (where .Site.RegularPages "Section" "products") }}
    {{ with (index . 0) }}
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
    {{ end }}
    {{ end }}
  </sitemap>
</sitemapindex>
```

**Note:** Rename default sitemap template to avoid conflict.

### Section-Specific Sitemaps

**Create sitemap for each section:**

**layouts/_default/sitemap-section.xml:**

```xml
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  {{ range .Data.Pages }}
  <url>
    <loc>{{ .Permalink }}</loc>
    {{ if not .Lastmod.IsZero }}
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
    {{ end }}
    <changefreq>weekly</changefreq>
    <priority>0.8</priority>
  </url>
  {{ end }}
</urlset>
```

**Use custom output format:**

**config.toml:**

```toml
[outputs]
  home = ["HTML", "RSS", "SITEMAP"]
  section = ["HTML", "RSS", "SITEMAP-SECTION"]

[outputFormats]
  [outputFormats.SITEMAP]
    mediaType = "application/xml"
    baseName = "sitemap"
    isPlainText = true

  [outputFormats.SITEMAP-SECTION]
    mediaType = "application/xml"
    baseName = "sitemap"
    isPlainText = true
```

### Dynamic Sitemap Index

**Generate sitemap index from all sections:**

**layouts/index.sitemap.xml:**

```xml
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  {{/* Homepage sitemap */}}
  <sitemap>
    <loc>{{ .Site.BaseURL }}sitemap-home.xml</loc>
    <lastmod>{{ .Site.LastChange.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
  </sitemap>

  {{/* Section sitemaps */}}
  {{ range .Site.Sections }}
  <sitemap>
    <loc>{{ .Site.BaseURL }}sitemap-{{ .Section }}.xml</loc>
    {{ with .Lastmod }}
    <lastmod>{{ .Format "2006-01-02T15:04:05-07:00" }}</lastmod>
    {{ end }}
  </sitemap>
  {{ end }}
</sitemapindex>
```

## Organizing Sitemaps

### By Section

**Best for:** Sites with distinct content types

**Structure:**
```
/sitemap.xml              (index)
/sitemap-blog.xml         (all blog posts)
/sitemap-docs.xml         (all documentation)
/sitemap-news.xml         (all news articles)
/sitemap-products.xml     (all products)
```

**Benefits:**
- Clear organization
- Easy to update specific sections
- Better analytics per section

### By Language

**Best for:** Multilingual sites

**Structure:**
```
/sitemap.xml              (index)
/en/sitemap.xml           (English content)
/es/sitemap.xml           (Spanish content)
/fr/sitemap.xml           (French content)
/de/sitemap.xml           (German content)
```

**Hugo generates this automatically for multilingual sites.**

### By Date

**Best for:** High-volume publishing (news sites)

**Structure:**
```
/sitemap.xml              (index)
/sitemap-2024.xml         (2024 content)
/sitemap-2025.xml         (2025 content)
/sitemap-2026.xml         (2026 content)
```

**Implementation:**

```xml
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  {{ range (seq 2024 2026) }}
  <sitemap>
    <loc>{{ $.Site.BaseURL }}sitemap-{{ . }}.xml</loc>
  </sitemap>
  {{ end }}
</sitemapindex>
```

**Individual year sitemap:**

```xml
{{ $year := .Params.year }}
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  {{ range where .Site.RegularPages "Date.Year" $year }}
  <url>
    <loc>{{ .Permalink }}</loc>
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
  </url>
  {{ end }}
</urlset>
```

### By Content Type

**Best for:** Mixed media sites

**Structure:**
```
/sitemap.xml              (index)
/sitemap-articles.xml     (articles)
/sitemap-videos.xml       (videos)
/sitemap-images.xml       (images)
/sitemap-products.xml     (products)
```

## Submission to Search Engines

### Submit Sitemap Index

**robots.txt:**

```
User-agent: *
Allow: /

Sitemap: https://example.com/sitemap.xml
```

**Google Search Console:**
1. Submit only the sitemap index URL
2. Google automatically discovers and crawls referenced sitemaps

**Bing Webmaster Tools:**
1. Submit sitemap index URL
2. Bing crawls all referenced sitemaps

### Submit Individual Sitemaps (Optional)

**You can submit individual sitemaps separately:**

```
Sitemap: https://example.com/sitemap.xml
Sitemap: https://example.com/sitemap-blog.xml
Sitemap: https://example.com/sitemap-docs.xml
```

**But it's not necessary if you submit the sitemap index.**

## Best Practices

### Organization

**✅ DO:**
- Use sitemap index for sites > 50,000 URLs
- Organize by section for clarity
- Use consistent naming (sitemap-{section}.xml)
- Include lastmod in sitemap index
- Keep sitemaps under 50,000 URLs each
- Keep sitemaps under 50MB each

**❌ DON'T:**
- Create sitemap index for small sites (< 10,000 URLs)
- Mix organization schemes
- Exceed 50,000 sitemaps in index
- Include non-existent sitemaps
- Forget to update sitemap index

### URL Requirements

**✅ DO:**
- Use absolute URLs (https://example.com/sitemap-blog.xml)
- Use HTTPS if site supports it
- Keep URL structure consistent
- Include all active sitemaps

**❌ DON'T:**
- Use relative URLs (/sitemap-blog.xml)
- Include redirected sitemap URLs
- List empty sitemaps

### Maintenance

**✅ DO:**
- Update lastmod when sitemap changes
- Regenerate sitemap index when adding sections
- Test all sitemap URLs are accessible
- Monitor in Google Search Console
- Automate generation with Hugo

**❌ DON'T:**
- Manually edit sitemap index (automate it)
- Forget to regenerate after structure changes
- Leave broken sitemap URLs

## Performance Optimization

### Lazy Loading Sitemaps

**For very large sites, generate sitemaps on-demand:**

**Netlify _redirects:**

```
/sitemap-blog.xml /.netlify/functions/sitemap-blog 200
/sitemap-docs.xml /.netlify/functions/sitemap-docs 200
```

**Serverless function generates sitemap dynamically.**

### Compression

**Compress large sitemaps (optional):**

```bash
gzip -c sitemap-blog.xml > sitemap-blog.xml.gz
```

**Reference compressed sitemaps:**

```xml
<sitemap>
  <loc>https://example.com/sitemap-blog.xml.gz</loc>
  <lastmod>2026-02-13T10:00:00+00:00</lastmod>
</sitemap>
```

**Note:** Google supports gzipped sitemaps.

### Caching

**Set appropriate cache headers:**

```
# Netlify headers (_headers file)
/sitemap*.xml
  Cache-Control: public, max-age=3600, s-maxage=3600
```

**Cache sitemaps for 1 hour, revalidate frequently.**

## Testing and Validation

### Validate Sitemap Index

**Check XML syntax:**

```bash
xmllint --noout sitemap.xml
```

**Verify all referenced sitemaps exist:**

```bash
curl -s https://example.com/sitemap.xml | grep -o '<loc>[^<]*</loc>' | sed 's/<[^>]*>//g' | while read url; do
  echo "Testing $url"
  curl -sI "$url" | head -1
done
```

### Google Search Console

**Monitor sitemap index:**
1. Go to "Sitemaps"
2. Submit sitemap index
3. Check "Discovered URLs"
4. Verify all sitemaps are processed

**Check coverage:**
- All referenced sitemaps should appear
- Check for errors

### Hugo Testing

**Build and check:**

```bash
hugo
ls -lh public/sitemap*.xml

# Check sitemap index
cat public/sitemap.xml

# Check individual sitemaps
cat public/sitemap-blog.xml
```

**Local testing:**

```bash
hugo server

# Test sitemap index
curl http://localhost:1313/sitemap.xml

# Test individual sitemaps
curl http://localhost:1313/sitemap-blog.xml
```

## Common Issues

### Sitemap Index Not Found

**Problem:** `/sitemap.xml` returns 404

**Solutions:**

1. **Ensure Hugo generates it:**
   ```bash
   hugo
   ls public/sitemap.xml
   ```

2. **Check baseURL:**
   ```toml
   baseURL = "https://example.com"
   ```

3. **For multilingual sites, Hugo generates sitemap index automatically**

### Individual Sitemaps Not Found

**Problem:** Referenced sitemaps return 404

**Solutions:**

1. **Verify sitemaps exist:**
   ```bash
   ls public/sitemap-*.xml
   ```

2. **Check URLs match filenames:**
   ```xml
   <!-- Ensure this matches actual file -->
   <loc>https://example.com/sitemap-blog.xml</loc>
   ```

3. **Regenerate sitemaps:**
   ```bash
   hugo --cleanDestinationDir
   ```

### Empty Sitemap Index

**Problem:** Sitemap index has no `<sitemap>` entries

**Causes:**
- No sections defined
- Custom template error
- No content in sections

**Solution:**

Check sections exist:

```bash
hugo list all
```

### Duplicate URLs Across Sitemaps

**Problem:** Same URL appears in multiple sitemaps

**Impact:** No negative impact, but inefficient

**Solution:**

Ensure sitemaps filter content correctly:

```xml
{{ range where .Site.RegularPages "Section" "blog" }}
  <!-- Only blog posts -->
{{ end }}
```

## Complete Examples

### Sitemap Index for Sections

**layouts/sitemap.xml:**

```xml
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  {{/* Blog sitemap */}}
  {{ with (where .Site.RegularPages "Section" "blog") }}
  <sitemap>
    <loc>{{ $.Site.BaseURL }}sitemap-blog.xml</loc>
    <lastmod>{{ (index . 0).Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
  </sitemap>
  {{ end }}

  {{/* Documentation sitemap */}}
  {{ with (where .Site.RegularPages "Section" "docs") }}
  <sitemap>
    <loc>{{ $.Site.BaseURL }}sitemap-docs.xml</loc>
    <lastmod>{{ (index . 0).Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
  </sitemap>
  {{ end }}

  {{/* News sitemap */}}
  {{ with (where .Site.RegularPages "Section" "news") }}
  <sitemap>
    <loc>{{ $.Site.BaseURL }}sitemap-news.xml</loc>
    <lastmod>{{ (index . 0).Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
  </sitemap>
  {{ end }}
</sitemapindex>
```

### Section Sitemap Template

**layouts/section/sitemap-blog.xml:**

```xml
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  {{ range where .Site.RegularPages "Section" "blog" }}
  <url>
    <loc>{{ .Permalink }}</loc>
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
    <changefreq>weekly</changefreq>
    <priority>0.8</priority>
  </url>
  {{ end }}
</urlset>
```

**Similar templates for:**
- `layouts/section/sitemap-docs.xml`
- `layouts/section/sitemap-news.xml`
- `layouts/section/sitemap-products.xml`

### Multilingual Sitemap Index

**Hugo generates this automatically, but for reference:**

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap>
    <loc>https://example.com/en/sitemap.xml</loc>
  </sitemap>
  <sitemap>
    <loc>https://example.com/es/sitemap.xml</loc>
  </sitemap>
  <sitemap>
    <loc>https://example.com/fr/sitemap.xml</loc>
  </sitemap>
  <sitemap>
    <loc>https://example.com/de/sitemap.xml</loc>
  </sitemap>
</sitemapindex>
```

## Guidelines

### Essential

**Minimum for sitemap index:**
- Use when site exceeds 50,000 URLs
- Reference all section sitemaps
- Use absolute HTTPS URLs
- Include in robots.txt
- Submit to Google Search Console

### Recommended

**For better organization:**
- Organize by logical sections
- Include lastmod for each sitemap
- Test all sitemap URLs
- Monitor in Search Console
- Automate generation with Hugo

### Advanced

**For maximum efficiency:**
- Compress large sitemaps (gzip)
- Organize by date for news sites
- Generate sitemaps on-demand (serverless)
- Separate image/video sitemaps
- Multiple language support

## Benefits

Scalability. Handle unlimited URLs across multiple sitemaps.

Organization. Clear structure by section, language, or date.

Efficiency. Search engines crawl targeted sections.

Flexibility. Add/remove sections without affecting others.

Analytics. Track indexing by section in Search Console.

## Related

- [sitemaps-robots-txt.md](./sitemaps-robots-txt.md) - Basic sitemap and robots.txt
- [canonical-urls.md](./canonical-urls.md) - Canonical URL specification
- [hreflang-tags.md](./hreflang-tags.md) - Multilingual site markup
- [../../01-hugo-basics/hugo-content-management.md](../01-hugo-basics/hugo-content-management.md) - Content organization
