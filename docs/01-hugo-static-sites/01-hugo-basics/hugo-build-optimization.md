# Hugo Build Optimization

Build performance, caching strategies, asset optimization, and production configuration.

## Why It Matters
- Hugo is fast by default (<1s for most sites) but asset processing and large sites can slow builds
- Proper caching avoids redundant work across incremental builds
- Production builds need minification, fingerprinting, and clean output

## Key Commands
```bash
# Development
hugo server                              # Live reload
hugo server -D --navigateToChanged       # Include drafts, navigate to edits

# Production
hugo --minify --environment production --cleanDestinationDir

# Profiling
hugo --templateMetrics --templateMetricsHints   # Find slow templates
```

## Build Configuration
```toml
# config.toml
disableKinds = ["taxonomy", "term"]   # Disable unused features
enableGitInfo = false                  # Skip unless needed
buildDrafts = false
buildFuture = false

[caches.images]
  dir = ":resourceDir/_gen"
  maxAge = -1                          # Never expire

[minify]
  minifyOutput = true
```

## Key Recommendations
- Use `partialCached` for partials with identical output across pages (headers, footers)
- Process resources (image resize, SCSS) **outside** loops -- assign to a variable first
- Use `.Summary` instead of `.Content` on list pages
- Limit page queries: `{{ range first 10 (where .Site.RegularPages "Type" "blog") }}`
- Use environment-specific config: `config/_default/`, `config/production/`, `config/development/`
- Cache Hugo resources in CI: cache `resources/` directory keyed on content hash

## Production Checklist
1. `hugo --minify` for HTML/CSS/JS minification
2. Fingerprint all assets (`| fingerprint`) for cache busting
3. `--cleanDestinationDir` to remove orphaned files
4. `--environment production` for prod-specific config
5. Validate output size: `du -sh public/`

## Pitfalls
- Don't process assets inside `{{ range }}` loops (reprocesses every iteration)
- Don't call `getJSON` to external APIs in loops (use data files or caching)
- Don't skip `--cleanDestinationDir` (orphaned files accumulate)
- Don't forget to set `[caches]` config for image and module caching
- Don't commit `resources/_gen/` to git (add to `.gitignore`)
