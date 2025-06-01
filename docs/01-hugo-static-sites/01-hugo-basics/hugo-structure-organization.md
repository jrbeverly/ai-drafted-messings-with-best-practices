# Hugo Structure & Organization

Standard Hugo project directory layout and conventions for maintainable sites.

## Why It Matters
- Hugo relies on directory conventions for template lookup, asset processing, and URL generation
- Correct structure eliminates configuration overhead and enables scalability
- Separating content, layouts, assets, and static files prevents build issues

## Directory Layout

```
my-hugo-site/
├── archetypes/      # Content templates for `hugo new`
├── assets/          # Source assets processed by Hugo Pipes (SCSS, JS)
├── content/         # Markdown content (directory = URL structure)
├── data/            # Structured data files (JSON, YAML, TOML)
├── layouts/         # Go HTML templates
├── static/          # Files copied as-is to public/ (favicons, fonts)
├── themes/          # Themes (optional)
├── config.toml      # Site configuration (or config/ directory)
└── go.mod           # Hugo modules (optional)
```

## Key Recommendations
- `_index.md` = section/list page; `index.md` (no underscore) = leaf bundle (single page)
- Use `assets/` for files needing processing (SCSS, JS); use `static/` for copy-as-is files
- Use `config/` directory with `_default/`, `production/`, `development/` for multi-environment builds
- Template lookup order: `layouts/{section}/` > `layouts/_default/` > `themes/{name}/layouts/`

## Quick Reference
```toml
# config/_default/config.toml
baseURL = "/"
title = "My Site"

# config/production/config.toml
baseURL = "https://example.com"
```

```bash
hugo server                        # Dev (default env)
hugo --environment production      # Production build
```

## Content Organization Patterns
- **Blog:** `content/blog/_index.md` + individual posts
- **Docs:** `content/docs/getting-started/_index.md` + nested sections
- **Multi-language:** `content/en/`, `content/es/` directories or `.en.md`, `.es.md` suffixes

## Gitignore Essentials
```gitignore
/public/
/resources/
.hugo_build.lock
node_modules/
```

## Pitfalls
- Don't commit `public/` or `resources/_gen/` (generated output)
- Don't nest content deeper than 3 levels (URLs become unwieldy)
- Don't mix processed and static assets in the same directory
- Don't use spaces or special characters in content filenames
