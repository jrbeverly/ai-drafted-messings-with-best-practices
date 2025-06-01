# Hugo Data Files

Structured data in JSON, YAML, or TOML for separating data from templates and integrating external APIs.

## Why It Matters
- Separates structured data (team members, pricing, navigation) from template logic
- Non-developers can edit JSON/YAML files without touching Go templates
- `getJSON` / `getCSV` pull external API data at build time

## Basic Usage
Place files in `data/` directory. Access via `.Site.Data.<filename>`.

```yaml
# data/navigation.yaml
main:
  - name: Home
    url: /
  - name: Blog
    url: /blog/
```
```go-html-template
{{ range .Site.Data.navigation.main }}
  <a href="{{ .url }}">{{ .name }}</a>
{{ end }}
```

## Common Use Cases
- **Team/authors:** `data/team.json` -- array of objects with name, role, photo, links
- **Pricing tables:** `data/pricing.yaml` -- plans with features and prices
- **Testimonials:** `data/testimonials.json` -- quotes, authors, ratings
- **FAQ:** `data/faq.yaml` -- categories with question/answer pairs
- **Navigation:** `data/navigation.yaml` -- menus with nested children

## Nested Data
```
data/
├── authors.json
├── products/
│   ├── featured.json
│   └── categories.yaml
```
Access: `{{ range .Site.Data.products.featured }}`

## External Data
```go-html-template
{{ $data := getJSON "https://api.example.com/data.json" }}
```
```toml
# Cache external API responses
[caches.getjson]
  maxAge = "1h"
```
Force refresh: `hugo --ignoreCache`

## Data Transformation
```go-html-template
{{ $featured := where .Site.Data.products "featured" true }}
{{ range sort .Site.Data.team "name" }}
```

## Pitfalls
- Don't nest data deeper than 3 levels (hard to maintain and access)
- Don't commit API keys in data files -- use site params or environment variables
- Don't fetch large datasets unnecessarily via `getJSON` (slows builds)
- Don't mix unrelated data types in one file
- Don't forget to validate JSON/YAML syntax before building (`jq .` or `python -c "import yaml; ..."`)
