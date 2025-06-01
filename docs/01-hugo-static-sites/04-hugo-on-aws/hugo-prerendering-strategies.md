# Hugo Prerendering Strategies

Build-time rendering. Static site generation. Incremental builds. On-demand rendering. Hybrid static/dynamic patterns.

## Principle

Hugo is a static site generator - all pages are prerendered at build time. Understand when full prerendering is sufficient, when incremental builds help, and when hybrid patterns with serverless functions complement Hugo's static output. Choose the right rendering strategy based on content volume, update frequency, and dynamic requirements.

## Rendering Strategy Overview

### Static Site Generation (Default)

```
Build Time                          Request Time
───────────                         ────────────
Hugo Build → HTML files → S3 → CloudFront → User gets cached HTML
```

**Every page built at deploy time.** This is Hugo's core model and the right choice for most sites.

### Hybrid: Static + API

```
Build Time                          Request Time
───────────                         ────────────
Hugo Build → HTML files → S3 ──→ CloudFront → Static HTML
                                      ↓
                          API Gateway → Lambda → Dynamic data (JSON)
```

**Static pages + client-side API calls** for dynamic content (comments, search, user data).

### On-Demand: Lambda@Edge Rendering

```
Request Time
────────────
CloudFront → Lambda@Edge → Generate HTML on request → Cache at edge
```

**Pages generated on first request.** Useful for millions of pages that rarely change.

## Full Prerendering (Hugo Default)

### When to Use

- Sites with < 50,000 pages
- Content changes via Git (editorial workflow)
- Build time < 5 minutes
- All content known at build time

### Hugo Build Performance

**Hugo is extremely fast:**

| Pages   | Typical Build Time |
| ------- | ------------------ |
| 100     | < 1 second         |
| 1,000   | 1-3 seconds        |
| 10,000  | 5-15 seconds       |
| 50,000  | 30-90 seconds      |
| 100,000 | 2-5 minutes        |

### Optimize Build Time

**config.toml:**

```toml
# Disable unused features
disableKinds = ["taxonomy", "term"]  # If not using taxonomies

# Limit image processing
[imaging]
  quality = 85
  resampleFilter = "Lanczos"

[imaging.exif]
  disableDate = true
  disableLatLong = true

# Parallel rendering (Hugo default)
[build]
  writeStats = false  # Disable unless needed for PurgeCSS
```

**Build command optimization:**

```bash
# Full optimized build
hugo --minify --gc --cleanDestinationDir

# Skip draft content
hugo --minify --gc --buildDrafts=false

# Limit to specific content (for testing)
hugo --minify --renderSegments blog
```

### Content Organization for Fast Builds

```
content/
├── blog/          # 500 posts - rendered
├── docs/          # 200 pages - rendered
├── pages/         # 20 static pages - rendered
└── data/          # Not rendered (used by templates)
```

**Avoid:**

- Deeply nested content hierarchies
- Excessive image processing in templates
- Complex taxonomy intersections
- Heavy use of `.GetPage` across sections

## Incremental Builds

### Hugo's Built-in Caching

**Hugo caches processed resources between builds:**

```bash
# Resources are cached in resources/_gen/
resources/
└── _gen/
    ├── assets/      # Processed CSS, JS, images
    └── images/      # Resized/cropped images
```

**Preserve cache in CI/CD:**

```yaml
# Gitea Actions / GitHub Actions
- name: Cache Hugo resources
  uses: actions/cache@v4
  with:
    path: resources/_gen
    key: hugo-resources-${{ hashFiles('assets/**', 'content/**/*.md') }}
    restore-keys: |
      hugo-resources-
```

### Content-Aware Deployment

**Only deploy changed files:**

```bash
# Hugo deploy only syncs changed files
hugo deploy --target production

# AWS CLI sync only uploads changed files (by size/timestamp)
aws s3 sync public/ s3://example-com-hugo/ --delete --size-only
```

### Partial Rebuild Pattern

**For very large sites, build only changed sections:**

```bash
#!/bin/bash
# partial-build.sh - Build only changed content sections

# Get changed files from last commit
CHANGED=$(git diff --name-only HEAD~1 HEAD -- content/)

# Determine affected sections
SECTIONS=""
for file in $CHANGED; do
  SECTION=$(echo "$file" | cut -d'/' -f2)
  SECTIONS="$SECTIONS $SECTION"
done
SECTIONS=$(echo "$SECTIONS" | tr ' ' '\n' | sort -u)

if [ -z "$SECTIONS" ]; then
  echo "No content changes, skipping build"
  exit 0
fi

echo "Changed sections: $SECTIONS"

# Build full site (Hugo doesn't support partial section builds natively)
# But we can use this info to only invalidate changed paths
hugo --minify --gc

# Invalidate only changed sections in CloudFront
PATHS=""
for section in $SECTIONS; do
  PATHS="$PATHS /${section}/*"
done

aws cloudfront create-invalidation \
  --distribution-id "$DISTRIBUTION_ID" \
  --paths $PATHS
```

## Hybrid Static + Dynamic

### Static HTML + Client-Side API

**Hugo generates static pages. JavaScript fetches dynamic data at runtime.**

**Hugo template (static shell):**

```go-html-template
{{ define "main" }}
<article>
  <h1>{{ .Title }}</h1>
  {{ .Content }}

  {{/* Static content above, dynamic below */}}
  <section id="comments" data-page-id="{{ .File.UniqueID }}">
    <h2>Comments</h2>
    <div id="comments-list">Loading comments...</div>
  </section>
</article>

{{/* Client-side script fetches comments from API */}}
<script>
  const pageId = document.getElementById('comments').dataset.pageId;
  fetch(`https://api.example.com/comments/${pageId}`)
    .then(r => r.json())
    .then(comments => {
      const list = document.getElementById('comments-list');
      list.innerHTML = comments.map(c =>
        `<div class="comment"><strong>${c.author}</strong><p>${c.text}</p></div>`
      ).join('');
    })
    .catch(() => {
      document.getElementById('comments-list').textContent = 'Comments unavailable.';
    });
</script>
{{ end }}
```

### Common Hybrid Patterns

| Static (Hugo)      | Dynamic (API/Lambda) |
| ------------------ | -------------------- |
| Blog posts, pages  | Comments             |
| Product listings   | Inventory/pricing    |
| Documentation      | Search               |
| Landing pages      | Contact forms        |
| Author profiles    | Newsletter signup    |
| Navigation, footer | User authentication  |

### Search Implementation

**Build search index at build time, query at runtime:**

**Hugo template to generate search index:**

```go-html-template
{{/* layouts/_default/index.json */}}
{{- $pages := where .Site.RegularPages "Type" "not in" (slice "page") -}}
[
  {{- range $index, $page := $pages -}}
    {{- if $index }},{{ end }}
    {
      "title": {{ .Title | jsonify }},
      "url": {{ .RelPermalink | jsonify }},
      "content": {{ .Plain | truncate 300 | jsonify }},
      "tags": {{ .Params.tags | jsonify }},
      "date": {{ .Date.Format "2006-01-02" | jsonify }},
      "section": {{ .Section | jsonify }}
    }
  {{- end }}
]
```

**config.toml:**

```toml
[outputs]
  home = ["HTML", "RSS", "JSON"]
```

**Client-side search with Fuse.js:**

```html
<script src="/js/fuse.min.js"></script>
<script>
  let searchIndex;

  fetch("/index.json")
    .then((r) => r.json())
    .then((data) => {
      searchIndex = new Fuse(data, {
        keys: ["title", "content", "tags"],
        threshold: 0.3,
      });
    });

  function search(query) {
    if (!searchIndex) return [];
    return searchIndex.search(query).map((r) => r.item);
  }
</script>
```

### Form Handling with Lambda

**Hugo contact form → API Gateway → Lambda → SES:**

```go-html-template
{{/* Hugo template - static form */}}
<form id="contact-form" action="https://api.example.com/contact" method="POST">
  <input type="text" name="name" required>
  <input type="email" name="email" required>
  <textarea name="message" required></textarea>
  <button type="submit">Send</button>
</form>

<script>
  document.getElementById('contact-form').addEventListener('submit', async (e) => {
    e.preventDefault();
    const form = e.target;
    const data = Object.fromEntries(new FormData(form));

    const response = await fetch(form.action, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });

    if (response.ok) {
      form.innerHTML = '<p>Thank you! Message sent.</p>';
    } else {
      alert('Failed to send. Please try again.');
    }
  });
</script>
```

## On-Demand Rendering

### When Full Prerendering Isn't Feasible

**Scenarios:**

- Millions of possible URL combinations
- Content from external APIs that changes frequently
- Personalized content per user/region
- Real-time data display

### Lambda@Edge On-Demand Rendering

**Generate Hugo-like pages on request:**

```javascript
// on-demand-render.mjs - Lambda@Edge origin-request
import {
  S3Client,
  GetObjectCommand,
  PutObjectCommand,
} from "@aws-sdk/client-s3";

const s3 = new S3Client({ region: "us-east-1" });
const BUCKET = "example-com-hugo";

export const handler = async (event) => {
  const request = event.Records[0].cf.request;
  const uri = request.uri;

  // Try S3 first (prerendered page exists)
  try {
    const key = uri.endsWith("/") ? uri.slice(1) + "index.html" : uri.slice(1);
    await s3.send(new GetObjectCommand({ Bucket: BUCKET, Key: key }));
    // File exists, let CloudFront serve it
    return request;
  } catch (e) {
    // File doesn't exist, render on demand
  }

  // Generate page on demand
  const html = await renderPage(uri);

  if (!html) {
    return {
      status: "404",
      statusDescription: "Not Found",
      body: "<html><body><h1>404 Not Found</h1></body></html>",
      headers: {
        "content-type": [{ key: "Content-Type", value: "text/html" }],
      },
    };
  }

  // Optionally cache to S3 for future requests
  const key = uri.endsWith("/") ? uri.slice(1) + "index.html" : uri.slice(1);
  await s3.send(
    new PutObjectCommand({
      Bucket: BUCKET,
      Key: key,
      Body: html,
      ContentType: "text/html",
      CacheControl: "public, max-age=3600",
    }),
  );

  return {
    status: "200",
    statusDescription: "OK",
    body: html,
    headers: {
      "content-type": [
        { key: "Content-Type", value: "text/html; charset=utf-8" },
      ],
      "cache-control": [
        { key: "Cache-Control", value: "public, max-age=3600" },
      ],
    },
  };
};

async function renderPage(uri) {
  // Fetch data and render HTML
  // This could use a template engine or simple string concatenation
  return `<!DOCTYPE html><html><body><h1>Generated: ${uri}</h1></body></html>`;
}
```

### Stale-While-Revalidate Pattern

**Serve cached content while regenerating in background:**

```
1. First request: Lambda generates page → cache in S3 + CloudFront
2. Subsequent requests: CloudFront serves cached page
3. Cache expires: CloudFront serves stale page + triggers background regeneration
4. Next request: Fresh page served
```

**Implementation with CloudFront cache policy:**

```hcl
resource "aws_cloudfront_cache_policy" "stale_while_revalidate" {
  name        = "hugo-swr"
  default_ttl = 3600       # 1 hour (serve cached version)
  max_ttl     = 86400      # 1 day maximum
  min_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    cookies_config { cookie_behavior = "none" }
    headers_config { header_behavior = "none" }
    query_strings_config { query_string_behavior = "none" }
  }
}
```

## Build Optimization Strategies

### Parallel Image Processing

```go-html-template
{{/* Process images in parallel using Hugo Pipes */}}
{{ $hero := resources.Get "images/hero.jpg" }}
{{ $thumb := $hero.Resize "300x" }}
{{ $medium := $hero.Resize "800x" }}
{{ $large := $hero.Resize "1200x" }}
{{ $webp := $hero.Resize "800x webp" }}

{{/* Hugo processes these in parallel automatically */}}
<picture>
  <source srcset="{{ $webp.RelPermalink }}" type="image/webp">
  <source srcset="{{ $large.RelPermalink }}" media="(min-width: 1200px)">
  <source srcset="{{ $medium.RelPermalink }}" media="(min-width: 800px)">
  <img src="{{ $thumb.RelPermalink }}" alt="Hero" loading="lazy">
</picture>
```

### Build Caching in CI

```yaml
# Cache Hugo module cache and processed resources
- name: Cache Hugo
  uses: actions/cache@v4
  with:
    path: |
      resources/_gen
      /tmp/hugo_cache
    key: hugo-${{ runner.os }}-${{ hashFiles('go.sum', 'assets/**') }}
    restore-keys: |
      hugo-${{ runner.os }}-

- name: Build
  env:
    HUGO_CACHEDIR: /tmp/hugo_cache
  run: hugo --minify --gc
```

### Content Segmentation

**For very large sites, split builds:**

```bash
# Build only blog section for blog-specific deploys
hugo --minify --renderSegments blog

# Build documentation separately
hugo --minify --renderSegments docs
```

**config.toml:**

```toml
# Define segments (Hugo 0.120+)
[segments.blog]
  [[segments.blog.includes]]
    kind = '{home,section,page}'
    path = '{/blog,/blog/**}'

[segments.docs]
  [[segments.docs.includes]]
    kind = '{home,section,page}'
    path = '{/docs,/docs/**}'
```

## Decision Matrix

### Which Strategy to Use?

| Factor                 | Full Prerender    | Incremental | Hybrid   | On-Demand |
| ---------------------- | ----------------- | ----------- | -------- | --------- |
| Pages < 10K            | Best              | Good        | Overkill | Overkill  |
| Pages 10K-100K         | Good              | Best        | Good     | Good      |
| Pages > 100K           | Slow builds       | Best        | Good     | Best      |
| Content changes rarely | Best              | Good        | Good     | Good      |
| Content changes hourly | Frequent rebuilds | Good        | Best     | Best      |
| User-specific content  | Cannot            | Cannot      | Best     | Good      |
| Real-time data         | Cannot            | Cannot      | Best     | Good      |
| Cost priority          | Cheapest          | Cheap       | Medium   | Higher    |
| Complexity             | Lowest            | Low         | Medium   | Highest   |

### Recommendation for solo developer

**Start with full prerendering.** Hugo builds are fast enough for most sites. Add hybrid patterns only when you have a specific need (comments, search, forms). On-demand rendering is rarely needed for Hugo sites.

## Best Practices

**DO:**

- Use Hugo's full static build as the default strategy
- Cache `resources/_gen` in CI for faster rebuilds
- Use asset fingerprinting for immutable caching
- Use Hugo's built-in deploy for smart S3 syncing
- Generate search indexes at build time
- Use client-side JavaScript for dynamic features

**DON'T:**

- Over-engineer rendering for a site Hugo can fully build
- Use on-demand rendering when prerendering is sufficient
- Skip build caching in CI/CD
- Process images in templates without caching
- Build the full site when only one section changed
- Add serverless complexity without a clear need

## Guidelines

### Essential

- Hugo `--minify --gc --cleanDestinationDir` for production
- Full prerendering for sites under 50K pages
- CI/CD automated builds on push
- Resource caching between builds

### Recommended

- Hugo deploy with smart sync
- Build caching in CI (resources/\_gen)
- Content segmentation for large sites
- Client-side search with prebuilt index
- Hybrid pattern for forms and comments

### Advanced

- On-demand rendering with Lambda@Edge
- Stale-while-revalidate caching
- A/B testing with edge functions
- Build segmentation for very large sites
- Background regeneration patterns

## Benefits

Speed. Hugo builds thousands of pages in seconds.

Simplicity. Static files need no runtime, no database, no server.

Reliability. Prerendered HTML never crashes, never times out.

Cost. S3 + CloudFront costs pennies per month.

Security. No attack surface on static HTML.

## Related

- [hugo-s3-deployment.md](./hugo-s3-deployment.md) - S3 deployment
- [hugo-cloudfront-integration.md](./hugo-cloudfront-integration.md) - CloudFront CDN
- [hugo-cicd-aws.md](./hugo-cicd-aws.md) - CI/CD pipeline
- [hugo-lambda-edge-rewrites.md](./hugo-lambda-edge-rewrites.md) - Edge functions
