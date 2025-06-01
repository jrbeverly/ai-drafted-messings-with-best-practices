# Sitemap.xml (Advanced)

XML sitemaps for search engines. Priority, change frequency, last modified. Sitemap index. Video, image, news sitemaps.

## Principle

Help search engines discover and crawl all pages on your site. Provide hints about page importance, update frequency, and content type. Accelerate indexing of new content.

## What is a Sitemap?

**Sitemap.xml:** XML file listing all URLs on your website

**Purpose:**
- Help search engines discover pages
- Indicate page importance (priority)
- Specify update frequency (changefreq)
- Show last modification date (lastmod)

**Location:** `/sitemap.xml` (root of domain)

**Specification:** https://www.sitemaps.org/protocol.html

**Supported By:** Google, Bing, Yandex, all major search engines

## Basic Sitemap

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com/</loc>
    <lastmod>2026-02-13</lastmod>
    <changefreq>daily</changefreq>
    <priority>1.0</priority>
  </url>

  <url>
    <loc>https://example.com/about</loc>
    <lastmod>2026-02-10</lastmod>
    <changefreq>monthly</changefreq>
    <priority>0.8</priority>
  </url>

  <url>
    <loc>https://example.com/blog/post-1</loc>
    <lastmod>2026-02-13T15:30:00Z</lastmod>
    <changefreq>weekly</changefreq>
    <priority>0.6</priority>
  </url>
</urlset>
```

## Sitemap Elements

### Required Elements

**`<urlset>`:** Root element
```xml
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <!-- URLs here -->
</urlset>
```

**`<url>`:** Container for each URL
```xml
<url>
  <loc>https://example.com/page</loc>
</url>
```

**`<loc>`:** Page URL (required)
```xml
<loc>https://example.com/page</loc>
```

- Must be absolute URL (https://)
- Max length: 2,048 characters
- Must be URL-encoded if contains special characters

### Optional Elements

**`<lastmod>`:** Last modification date

```xml
<lastmod>2026-02-13</lastmod>
<!-- or with time -->
<lastmod>2026-02-13T15:30:00Z</lastmod>
```

- Format: ISO 8601 (YYYY-MM-DD or YYYY-MM-DDTHH:MM:SSZ)
- Timezone: Z for UTC, or +/-HH:MM offset

**`<changefreq>`:** Update frequency hint

```xml
<changefreq>daily</changefreq>
```

- Values: `always`, `hourly`, `daily`, `weekly`, `monthly`, `yearly`, `never`
- Hint only (search engines may ignore)
- **Reality:** Google mostly ignores this, but include anyway

**`<priority>`:** Relative importance

```xml
<priority>0.8</priority>
```

- Range: 0.0 to 1.0
- Default: 0.5
- Relative to other pages on YOUR site (not globally)
- **Reality:** Google mostly ignores this, but include anyway

## Hugo Default Sitemap

**Hugo automatically generates sitemap.xml**

**URL:** `/sitemap.xml`

**Default template:** Hugo's built-in sitemap

**Override:** Create `layouts/sitemap.xml`

### Hugo Built-in Sitemap

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  {{ range .Data.Pages }}
  <url>
    <loc>{{ .Permalink }}</loc>
    {{ if not .Lastmod.IsZero }}
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05-07:00" }}</lastmod>
    {{ end }}
  </url>
  {{ end }}
</urlset>
```

**Customization:** Use Hugo configuration or custom template

## Hugo Sitemap Configuration

### Basic Configuration (config.toml)

```toml
[sitemap]
changefreq = "monthly"
filename = "sitemap.xml"
priority = 0.5
```

### Per-Section Configuration

```toml
[sitemap]
changefreq = "monthly"
priority = 0.5

# Override for specific sections
[[menu.main]]
[menu.main.params]
  changefreq = "daily"
  priority = 1.0
```

### Front Matter Override

```yaml
---
title: "Important Page"
sitemap:
  changefreq: daily
  priority: 1.0
---
```

## Custom Hugo Sitemap Template

```go-html-template
{{/* layouts/sitemap.xml */}}
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  {{- range .Data.Pages }}
  {{- if not .Params.noindex }}
  <url>
    <loc>{{ .Permalink }}</loc>

    {{- if not .Lastmod.IsZero }}
    <lastmod>{{ .Lastmod.Format "2006-01-02T15:04:05Z07:00" }}</lastmod>
    {{- end }}

    {{- with .Params.sitemap.changefreq }}
    <changefreq>{{ . }}</changefreq>
    {{- else }}
    {{- with $.Site.Sitemap.ChangeFreq }}
    <changefreq>{{ . }}</changefreq>
    {{- end }}
    {{- end }}

    {{- if isset .Params.sitemap "priority" }}
    <priority>{{ .Params.sitemap.priority }}</priority>
    {{- else }}
    {{- with $.Site.Sitemap.Priority }}
    <priority>{{ . }}</priority>
    {{- end }}
    {{- end }}
  </url>
  {{- end }}
  {{- end }}
</urlset>
```

## Sitemap Index

**Problem:** Sitemaps limited to 50,000 URLs or 50 MB

**Solution:** Sitemap index file referencing multiple sitemaps

### Sitemap Index Format

```xml
<?xml version="1.0" encoding="UTF-8"?>
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap>
    <loc>https://example.com/sitemap-posts.xml</loc>
    <lastmod>2026-02-13T10:00:00Z</lastmod>
  </sitemap>

  <sitemap>
    <loc>https://example.com/sitemap-pages.xml</loc>
    <lastmod>2026-02-10T12:00:00Z</lastmod>
  </sitemap>

  <sitemap>
    <loc>https://example.com/sitemap-products.xml</loc>
    <lastmod>2026-02-12T15:30:00Z</lastmod>
  </sitemap>
</sitemapindex>
```

### Hugo Sitemap Index

```go-html-template
{{/* layouts/sitemapindex.xml */}}
{{ printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"?>" | safeHTML }}
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap>
    <loc>{{ "/sitemap-posts.xml" | absURL }}</loc>
    <lastmod>{{ now.Format "2006-01-02T15:04:05Z07:00" }}</lastmod>
  </sitemap>

  <sitemap>
    <loc>{{ "/sitemap-pages.xml" | absURL }}</loc>
    <lastmod>{{ now.Format "2006-01-02T15:04:05Z07:00" }}</lastmod>
  </sitemap>
</sitemapindex>
```

**Configure in Hugo:**

```toml
[outputs]
home = ["HTML", "RSS", "SitemapIndex"]

[outputFormats]
[outputFormats.SitemapIndex]
baseName = "sitemap"
mediaType = "application/xml"
```

## Video Sitemap

**Purpose:** Help Google discover video content

**Namespace:** `http://www.google.com/schemas/sitemap-video/1.1`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
        xmlns:video="http://www.google.com/schemas/sitemap-video/1.1">
  <url>
    <loc>https://example.com/videos/tutorial</loc>

    <video:video>
      <video:thumbnail_loc>https://example.com/thumbs/tutorial.jpg</video:thumbnail_loc>
      <video:title>How to Use Schema.org</video:title>
      <video:description>Complete guide to Schema.org structured data</video:description>
      <video:content_loc>https://example.com/videos/tutorial.mp4</video:content_loc>
      <video:player_loc>https://example.com/embed/tutorial</video:player_loc>
      <video:duration>600</video:duration>
      <video:publication_date>2026-02-13T10:00:00Z</video:publication_date>
      <video:family_friendly>yes</video:family_friendly>
      <video:rating>4.8</video:rating>
      <video:view_count>12847</video:view_count>
    </video:video>
  </url>
</urlset>
```

**Required fields:**
- `thumbnail_loc`: Thumbnail URL
- `title`: Video title
- `description`: Video description
- `content_loc` OR `player_loc`: Video file or player URL

**Optional fields:**
- `duration`: Length in seconds
- `publication_date`: Upload date
- `family_friendly`: yes/no
- `rating`: 0.0 to 5.0
- `view_count`: Number of views

## Image Sitemap

**Purpose:** Help Google discover images

**Namespace:** `http://www.google.com/schemas/sitemap-image/1.1`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
        xmlns:image="http://www.google.com/schemas/sitemap-image/1.1">
  <url>
    <loc>https://example.com/blog/post</loc>

    <image:image>
      <image:loc>https://example.com/images/photo1.jpg</image:loc>
      <image:caption>Beautiful sunset over mountains</image:caption>
      <image:geo_location>San Francisco, CA</image:geo_location>
      <image:title>Sunset Mountains</image:title>
      <image:license>https://example.com/license</image:license>
    </image:image>

    <image:image>
      <image:loc>https://example.com/images/photo2.jpg</image:loc>
      <image:caption>City skyline at night</image:caption>
    </image:image>
  </url>
</urlset>
```

**Required:**
- `image:loc`: Image URL

**Optional:**
- `image:caption`: Image description
- `image:geo_location`: Location
- `image:title`: Image title
- `image:license`: License URL

## News Sitemap

**Purpose:** For news publishers (Google News)

**Namespace:** `http://www.google.com/schemas/sitemap-news/0.9`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
        xmlns:news="http://www.google.com/schemas/sitemap-news/0.9">
  <url>
    <loc>https://example.com/news/article-1</loc>

    <news:news>
      <news:publication>
        <news:name>Example News</news:name>
        <news:language>en</news:language>
      </news:publication>

      <news:publication_date>2026-02-13T10:00:00Z</news:publication_date>
      <news:title>Breaking News: Important Event</news:title>

      <news:keywords>politics, economy, breaking news</news:keywords>
    </news:news>
  </url>
</urlset>
```

**Required:**
- `publication/name`: Publication name
- `publication/language`: Language code
- `publication_date`: Article date (last 2 days for Google News)
- `title`: Article title

**Note:** Only for verified news publishers in Google News.

## Sitemap Best Practices

### Priority Recommendations

```xml
<!-- Homepage -->
<url>
  <loc>https://example.com/</loc>
  <priority>1.0</priority>
  <changefreq>daily</changefreq>
</url>

<!-- Important pages (about, contact) -->
<url>
  <loc>https://example.com/about</loc>
  <priority>0.8</priority>
  <changefreq>monthly</changefreq>
</url>

<!-- Blog posts, articles -->
<url>
  <loc>https://example.com/blog/post</loc>
  <priority>0.6</priority>
  <changefreq>weekly</changefreq>
</url>

<!-- Archive pages, categories -->
<url>
  <loc>https://example.com/blog</loc>
  <priority>0.5</priority>
  <changefreq>daily</changefreq>
</url>

<!-- Tags, less important pages -->
<url>
  <loc>https://example.com/tags/seo</loc>
  <priority>0.3</priority>
  <changefreq>weekly</changefreq>
</url>
```

### Exclude Pages from Sitemap

```go-html-template
{{/* Hugo - exclude noindex pages */}}
{{ range .Data.Pages }}
  {{ if not .Params.noindex }}
  <url>
    <loc>{{ .Permalink }}</loc>
  </url>
  {{ end }}
{{ end }}
```

**Front matter:**

```yaml
---
title: "Private Page"
noindex: true  # Won't appear in sitemap
---
```

## Submit Sitemap to Search Engines

### Google Search Console

1. Go to https://search.google.com/search-console
2. Select property
3. Sitemaps (left sidebar)
4. Enter sitemap URL: `https://example.com/sitemap.xml`
5. Click "Submit"

### Bing Webmaster Tools

1. Go to https://www.bing.com/webmasters
2. Select site
3. Sitemaps
4. Submit sitemap URL

### Robots.txt Reference

```txt
# robots.txt
User-agent: *
Allow: /

Sitemap: https://example.com/sitemap.xml
```

**Hugo robots.txt template:**

```go-html-template
{{/* layouts/robots.txt */}}
User-agent: *
{{ if eq (getenv "HUGO_ENV") "production" }}
Allow: /
{{ else }}
Disallow: /
{{ end }}

Sitemap: {{ .Site.BaseURL }}sitemap.xml
```

## Testing Sitemap

### Validate XML

```bash
# Check if sitemap exists
curl -I https://example.com/sitemap.xml
# Should return 200 OK

# Download and view
curl https://example.com/sitemap.xml

# Validate XML syntax
curl https://example.com/sitemap.xml | xmllint --format -

# Count URLs
curl https://example.com/sitemap.xml | grep -c "<loc>"
```

### Google Search Console

**URL:** https://search.google.com/search-console

**Sitemaps Report:**
- Shows submitted sitemaps
- Discovered URLs
- Errors and warnings
- Last read date

**Common errors:**
- Invalid XML
- HTTP errors (404, 500)
- URL errors (redirects, blocked by robots.txt)

### Online Validators

**XML Sitemap Validator:** https://www.xml-sitemaps.com/validate-xml-sitemap.html

## Hugo Automation

### Auto-generate Sitemap on Build

Hugo automatically generates sitemap.xml on every build.

```bash
# Build site
hugo

# Verify sitemap
cat public/sitemap.xml
```

### CI/CD Integration

```yaml
# .github/workflows/deploy.yml
- name: Build Hugo site
  run: hugo --minify

- name: Verify sitemap
  run: |
    if [ ! -f public/sitemap.xml ]; then
      echo "ERROR: sitemap.xml not generated"
      exit 1
    fi

    # Count URLs
    URL_COUNT=$(grep -c "<loc>" public/sitemap.xml)
    echo "Sitemap contains $URL_COUNT URLs"

- name: Deploy to S3
  run: |
    aws s3 sync public/ s3://${{ secrets.S3_BUCKET }}/ \
      --delete \
      --cache-control "public, max-age=3600"

- name: Invalidate CloudFront
  run: |
    aws cloudfront create-invalidation \
      --distribution-id ${{ secrets.CLOUDFRONT_ID }} \
      --paths "/sitemap.xml"
```

### Ping Search Engines (Optional)

```bash
# Ping Google
curl "https://www.google.com/ping?sitemap=https://example.com/sitemap.xml"

# Ping Bing
curl "https://www.bing.com/ping?sitemap=https://example.com/sitemap.xml"
```

**Note:** Submitting via Search Console is preferred.

## Common Mistakes

❌ **Relative URLs:**
```xml
<!-- Wrong -->
<loc>/blog/post</loc>

<!-- Correct -->
<loc>https://example.com/blog/post</loc>
```

❌ **Including noindex pages:**
```xml
<!-- Wrong - page has noindex meta tag but appears in sitemap -->
<url>
  <loc>https://example.com/private</loc>
</url>
```

❌ **Including redirected URLs:**
```xml
<!-- Wrong - this URL redirects to another page -->
<url>
  <loc>https://example.com/old-url</loc>
</url>
```

❌ **Including blocked URLs:**
```xml
<!-- Wrong - URL blocked by robots.txt -->
<url>
  <loc>https://example.com/admin</loc>
</url>
```

❌ **Too many URLs:**
```xml
<!-- Wrong - sitemap has 60,000 URLs (max 50,000) -->
```

**Solution:** Use sitemap index with multiple sitemaps.

❌ **Invalid XML:**
```xml
<!-- Wrong - special characters not escaped -->
<loc>https://example.com/page?q=test&sort=date</loc>

<!-- Correct - & escaped as &amp; -->
<loc>https://example.com/page?q=test&amp;sort=date</loc>
```

## Sitemap Limits

**Maximum URLs:** 50,000 per sitemap
**Maximum file size:** 50 MB (uncompressed)
**Solution:** Use sitemap index if exceeding limits

**Compression:** Can gzip sitemap (reduces size)

```bash
# Compress sitemap
gzip -c sitemap.xml > sitemap.xml.gz

# Upload both versions
# Search engines support .xml.gz
```

## S3 Deployment

```bash
# Upload sitemap to S3
aws s3 cp public/sitemap.xml s3://your-bucket/sitemap.xml \
  --content-type "application/xml; charset=utf-8" \
  --cache-control "public, max-age=3600"

# Invalidate CloudFront cache
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/sitemap.xml"
```

**Terraform:**

```hcl
resource "aws_s3_bucket_object" "sitemap" {
  bucket       = aws_s3_bucket.website.id
  key          = "sitemap.xml"
  source       = "public/sitemap.xml"
  content_type = "application/xml; charset=utf-8"
  cache_control = "public, max-age=3600"  # 1 hour
  etag         = filemd5("public/sitemap.xml")
}
```

## Guidelines

**Required:**
- Valid XML format
- Absolute URLs (https://)
- `<loc>` for every URL
- Under 50,000 URLs per file
- Under 50 MB per file

**Recommended:**
- Include `<lastmod>` (helps search engines)
- Set appropriate `<priority>` values
- Include `<changefreq>` hints
- Exclude noindex pages
- Exclude redirected URLs
- Submit to Search Console

**Best Practices:**
- Update sitemap on content changes
- Reference in robots.txt
- Use sitemap index for large sites
- Monitor Search Console for errors
- Cache sitemap (1 hour TTL)

## Benefits

Discoverable. Search engines find all pages.

Indexed. Faster indexing of new content.

Organized. Structured view of site hierarchy.

Informative. Hints about importance and freshness.

## Related

- [robots-txt-advanced.md](./robots-txt-advanced.md) - robots.txt
- [canonical-urls.md](./canonical-urls.md) - Canonical tags
- [hreflang-tags.md](./hreflang-tags.md) - Hreflang in sitemaps
- [meta-tags-seo.md](./meta-tags-seo.md) - SEO meta tags
