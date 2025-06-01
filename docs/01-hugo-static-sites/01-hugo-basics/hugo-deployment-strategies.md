# Hugo Deployment Strategies

Static hosting platforms, CI/CD pipelines, AWS S3+CloudFront, and cache header configuration.

## Why It Matters
- Hugo outputs pure static files -- no server runtime needed, infinite scaling
- Correct cache headers (immutable for fingerprinted assets, no-cache for HTML) are critical
- CI/CD ensures consistent, automated, atomic deployments

## Build Command
```bash
hugo --minify --environment production --cleanDestinationDir
```

## Platform Comparison

| Platform | Pros | Setup |
|----------|------|-------|
| **Netlify** | Git-based deploys, preview deploys, headers/redirects | `netlify.toml` |
| **Vercel** | Fast edge network, zero-config Hugo support | `vercel.json` |
| **GitHub Pages** | Free, integrated with GitHub Actions | `.github/workflows/deploy.yml` |
| **AWS S3+CloudFront** | Full control, cost-effective at scale | Terraform + deploy script |

## Netlify Quick Setup
```toml
# netlify.toml
[build]
  publish = "public"
  command = "hugo --minify --environment production"
[build.environment]
  HUGO_VERSION = "0.119.0"
```

## GitHub Actions
```yaml
- uses: peaceiris/actions-hugo@v2
  with:
    hugo-version: '0.119.0'
    extended: true
- run: hugo --minify --environment production
```

## AWS S3 Deploy Pattern
```bash
# Fingerprinted assets: 1-year cache
aws s3 sync public/ s3://bucket --delete \
  --cache-control "public, max-age=31536000, immutable" \
  --exclude "*.html" --exclude "*.xml"
# HTML: always revalidate
aws s3 sync public/ s3://bucket \
  --cache-control "public, max-age=0, must-revalidate" \
  --exclude "*" --include "*.html" --include "*.xml"
# Invalidate CDN
aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/*"
```

## Cache Headers
- **Fingerprinted assets (CSS, JS, images):** `max-age=31536000, immutable`
- **HTML/XML:** `max-age=0, must-revalidate`

## Security Headers
```
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
```

## CI/CD Tips
- Cache `resources/` directory (processed images/SCSS) for faster builds
- Cache Hugo modules: key on `go.sum` hash
- Use `--cleanDestinationDir` to remove orphaned files
- Set `HUGO_ENV=production` as environment variable

## Pitfalls
- Don't forget to set `baseURL` correctly for the target environment
- Don't serve HTML with long cache headers (users will see stale content)
- Don't skip CloudFront invalidation after S3 deploy
- Don't store AWS credentials in code -- use CI/CD secrets or IAM roles
- Don't deploy without testing the build locally first
