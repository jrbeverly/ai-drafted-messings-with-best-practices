# Geo Meta Tags

Geographic metadata. Location-based SEO. Geo targeting. ICBM coordinates. Local search optimization.

## Principle

Use geo meta tags to associate your content with geographic locations. Improve local search visibility. Help search engines understand geographic relevance of your content.

## What are Geo Meta Tags?

**Geo meta tags:** HTML meta tags that specify geographic location of a page or business

**Purpose:**
- Local SEO optimization
- Geographic targeting
- Map integration
- Regional content association

**Use cases:**
- Local business websites
- Location-specific content
- Regional news/events
- Travel and tourism sites
- Multi-location organizations

## Geo Meta Tag Syntax

### Basic Geo Tags

```html
<meta name="geo.region" content="US-CA">
<meta name="geo.placename" content="San Francisco">
<meta name="geo.position" content="37.7749;-122.4194">
<meta name="ICBM" content="37.7749, -122.4194">
```

### All Geo Elements

**geo.region** - ISO 3166-1/2 region code

```html
<!-- Country -->
<meta name="geo.region" content="US">

<!-- Country + State/Province -->
<meta name="geo.region" content="US-CA">
<meta name="geo.region" content="GB-LND">
<meta name="geo.region" content="DE-BY">
```

**geo.placename** - Human-readable place name

```html
<meta name="geo.placename" content="San Francisco, California">
```

**geo.position** - Latitude and longitude (semicolon-separated)

```html
<meta name="geo.position" content="37.7749;-122.4194">
```

**ICBM** - Internet Content Based on Metadata (comma-separated coordinates)

```html
<meta name="ICBM" content="37.7749, -122.4194">
```

## Hugo Implementation

### Basic Template

**layouts/partials/head/geo-meta.html:**

```go-html-template
{{/* Geo meta tags */}}

{{ with .Params.geo_region }}
  <meta name="geo.region" content="{{ . }}">
{{ else }}
  {{ with .Site.Params.geo_region }}
    <meta name="geo.region" content="{{ . }}">
  {{ end }}
{{ end }}

{{ with .Params.geo_placename }}
  <meta name="geo.placename" content="{{ . }}">
{{ else }}
  {{ with .Site.Params.geo_placename }}
    <meta name="geo.placename" content="{{ . }}">
  {{ end }}
{{ end }}

{{ with .Params.geo_position }}
  <meta name="geo.position" content="{{ . }}">
  <meta name="ICBM" content="{{ replace . ";" ", " }}">
{{ else }}
  {{ with .Site.Params.geo_position }}
    <meta name="geo.position" content="{{ . }}">
    <meta name="ICBM" content="{{ replace . ";" ", " }}">
  {{ end }}
{{ end }}
```

### Content Front Matter

**Per-page geographic targeting:**

```yaml
---
title: "Hugo Meetup San Francisco"
geo_region: "US-CA"
geo_placename: "San Francisco, California"
geo_position: "37.7749;-122.4194"
---
```

### Site Configuration

**config.toml:**

```toml
[params]
  # Default geo tags for entire site
  geo_region = "US-CA"
  geo_placename = "San Francisco, California"
  geo_position = "37.7749;-122.4194"
```

### Multi-Location

**For businesses with multiple locations:**

```go-html-template
{{ with .Params.locations }}
  {{/* Use primary location for meta tags */}}
  {{ with index . 0 }}
    <meta name="geo.region" content="{{ .region }}">
    <meta name="geo.placename" content="{{ .placename }}">
    <meta name="geo.position" content="{{ .position }}">
    <meta name="ICBM" content="{{ replace .position ";" ", " }}">
  {{ end }}
{{ end }}
```

**Front matter:**

```yaml
---
title: "Our Locations"
locations:
  - region: "US-CA"
    placename: "San Francisco, CA"
    position: "37.7749;-122.4194"
  - region: "US-NY"
    placename: "New York, NY"
    position: "40.7128;-74.0060"
---
```

## Region Codes

### ISO 3166-1 (Country)

| Code | Country |
|------|---------|
| US | United States |
| GB | United Kingdom |
| CA | Canada |
| DE | Germany |
| FR | France |
| JP | Japan |
| AU | Australia |
| BR | Brazil |

### ISO 3166-2 (Country-Subdivision)

**United States:**

| Code | State |
|------|-------|
| US-CA | California |
| US-NY | New York |
| US-TX | Texas |
| US-WA | Washington |

**United Kingdom:**

| Code | Region |
|------|--------|
| GB-LND | London |
| GB-EDH | Edinburgh |
| GB-MAN | Manchester |

## Geo Tags with Schema.org

**Combine geo meta tags with Schema.org LocalBusiness:**

```html
<!-- Geo meta tags -->
<meta name="geo.region" content="US-CA">
<meta name="geo.placename" content="San Francisco">
<meta name="geo.position" content="37.7749;-122.4194">

<!-- Schema.org (more powerful for search engines) -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "LocalBusiness",
  "name": "Hugo Web Studio",
  "address": {
    "@type": "PostalAddress",
    "streetAddress": "123 Main St",
    "addressLocality": "San Francisco",
    "addressRegion": "CA",
    "postalCode": "94102",
    "addressCountry": "US"
  },
  "geo": {
    "@type": "GeoCoordinates",
    "latitude": "37.7749",
    "longitude": "-122.4194"
  }
}
</script>
```

**Note:** Schema.org LocalBusiness is more effective for local SEO than geo meta tags alone.

## Best Practices

**✅ DO:**
- Use ISO 3166 region codes
- Include both geo.position and ICBM
- Combine with Schema.org LocalBusiness
- Use for location-specific content
- Specify precise coordinates
- Include geo.placename for readability

**❌ DON'T:**
- Use geo tags for non-location content
- Specify wrong coordinates
- Use non-standard region codes
- Rely solely on geo tags for local SEO
- Include geo tags on every page (only location-relevant ones)

## Guidelines

### Essential

**Minimum for location content:**
- geo.region (ISO 3166 code)
- geo.placename (human-readable)
- geo.position (coordinates)

### Recommended

**For better local SEO:**
- Schema.org LocalBusiness
- Google Business Profile
- Consistent NAP (Name, Address, Phone)
- ICBM tag
- Combine with Open Graph locale

### Advanced

**For maximum local visibility:**
- Multi-location support
- Dynamic geo tags per page
- Combined with hreflang for regional targeting
- Integration with Google Maps
- Location-specific structured data

## Benefits

Local SEO. Improved visibility in location-based searches.

Geographic Context. Search engines understand content relevance to location.

Map Integration. Coordinates enable map-based discovery.

Regional Targeting. Content associated with specific regions.

## Related

- [structured-data-schema-org.md](./structured-data-schema-org.md) - Schema.org LocalBusiness
- [hreflang-tags.md](./hreflang-tags.md) - Regional language targeting
- [meta-tags-comprehensive.md](./meta-tags-comprehensive.md) - All HTML meta tags
- [open-graph-meta-tags.md](./open-graph-meta-tags.md) - og:locale for region
