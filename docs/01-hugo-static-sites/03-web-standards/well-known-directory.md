# .well-known Directory

Standardized location (RFC 8615) for site-wide metadata and machine-readable configuration files.

## Why It Matters

- Provides a **single discoverable location** for security, auth, and service metadata
- Used by browsers, crawlers, certificate authorities, and federation protocols
- Required by Let's Encrypt (ACME), iOS universal links, and security.txt

## Common .well-known Files

| File | Purpose |
|------|---------|
| `security.txt` | Security disclosure policy (RFC 9116) |
| `change-password` | Redirect to password reset form |
| `acme-challenge/` | Let's Encrypt certificate validation |
| `apple-app-site-association` | iOS universal links |
| `assetlinks.json` | Android App Links |
| `openid-configuration` | OAuth/OIDC discovery |
| `webfinger` | Federation/ActivityPub user discovery |
| `nodeinfo` | Fediverse server metadata |

## Hugo Setup

```yaml
# config.yaml — use a separate static mount
module:
  mounts:
    - source: static
      target: static
    - source: well-known
      target: static/.well-known
```

Or simply place files in `static/.well-known/` (Hugo copies them to `public/.well-known/`).

## Key Configuration

- **MIME types:** `text/plain` for `.txt`, `application/json` for `.json`
- **Caching:** Short TTL (1-24 hours) -- these files change occasionally
- **Security:** Disable directory listing, serve only intended files, require HTTPS

## CloudFront Cache Behavior

```hcl
ordered_cache_behavior {
  path_pattern           = "/.well-known/*"
  viewer_protocol_policy = "redirect-to-https"
  default_ttl            = 3600   # 1 hour
  max_ttl                = 86400  # 24 hours
}
```

## Pitfalls to Avoid

- Do not enable directory listing on `/.well-known/`
- Do not cache ACME challenges (must be fresh)
- Do not forget HTTPS -- sensitive files require it
- Do not serve with wrong Content-Type (match file format)

## Related

- [security-txt.md](./security-txt.md)
- [webmanifest-pwa.md](./webmanifest-pwa.md)
