# OpenSearch XML

XML descriptor that lets browsers add your site's search to their address bar / search dropdown.

## Why It Matters

- Users can search your site directly from Chrome/Firefox/Edge address bar
- Browsers auto-discover it via the `<link>` tag and offer "Add search engine"
- Supports autocomplete suggestions via a JSON endpoint

## Minimal opensearch.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<OpenSearchDescription xmlns="http://a9.com/-/spec/opensearch/1.1/">
  <ShortName>Example</ShortName>
  <Description>Search Example website</Description>
  <Url type="text/html" template="https://example.com/search?q={searchTerms}"/>
  <Image width="16" height="16" type="image/x-icon">https://example.com/favicon.ico</Image>
</OpenSearchDescription>
```

## HTML Discovery Tag (Required)

```html
<link rel="search" type="application/opensearchdescription+xml"
      title="Search Example" href="/opensearch.xml">
```

Without this tag in `<head>`, browsers will not detect your search engine.

## Key Elements

| Element | Required | Notes |
|---------|----------|-------|
| `<ShortName>` | Yes | Max 16 characters |
| `<Description>` | Yes | Max 1024 characters |
| `<Url type="text/html">` | Yes | Must include `{searchTerms}` |
| `<Image>` | Recommended | 16x16 favicon |

## Optional: Autocomplete Suggestions

```xml
<Url type="application/x-suggestions+json"
     template="https://example.com/api/suggest?q={searchTerms}"/>
```

Response format: `["query", ["suggestion1", "suggestion2"], ["desc1", "desc2"], ["url1", "url2"]]`

## Hugo Setup

Place at `static/opensearch.xml` or generate dynamically:

```yaml
# config.yaml
[outputFormats.OpenSearch]
  baseName = "opensearch"
  mediaType = "application/opensearchdescription+xml"
  isPlainText = true
```

## Serving Requirements

- **MIME type:** `application/opensearchdescription+xml; charset=utf-8`
- **Cache:** 1 day

## Pitfalls to Avoid

- `<ShortName>` exceeding 16 characters (truncated or ignored)
- Missing `{searchTerms}` in URL template (search will not work)
- Using `&` instead of `&amp;` in XML URL templates
- Forgetting the `<link rel="search">` discovery tag in HTML
- Wrong MIME type (`text/xml` instead of `application/opensearchdescription+xml`)

## Related

- [favicon-modern-formats.md](./favicon-modern-formats.md)
