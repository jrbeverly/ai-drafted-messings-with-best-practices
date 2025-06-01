# Content Security Policy (CSP)

HTTP header defining which resources (scripts, styles, images) can load and execute on your site. Primary defense against XSS.

## Why It Matters

- Blocks unauthorized scripts from executing (XSS prevention)
- Granular control per resource type (scripts, styles, images, fonts, connections)
- Violation reporting lets you detect attacks and misconfigurations

## Essential CSP

```
Content-Security-Policy: default-src 'self'; frame-ancestors 'none'
```

## Recommended CSP

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{random}';
  style-src 'self' 'nonce-{random}';
  img-src 'self' data: https:;
  font-src 'self';
  connect-src 'self';
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
  upgrade-insecure-requests
```

## Key Directives

| Directive | Controls |
|-----------|----------|
| `default-src` | Fallback for all resource types |
| `script-src` | JavaScript sources |
| `style-src` | CSS sources |
| `img-src` | Image sources |
| `font-src` | Font sources |
| `connect-src` | AJAX, WebSocket, fetch targets |
| `frame-src` | iframe sources |
| `frame-ancestors` | Who can embed your site (replaces X-Frame-Options) |
| `base-uri` | Restricts `<base>` element |
| `form-action` | Restricts form submission targets |

## Source Values

| Value | Meaning |
|-------|---------|
| `'none'` | Block everything |
| `'self'` | Same origin only |
| `https://cdn.example.com` | Specific domain |
| `https:` | Any HTTPS source |
| `data:` | Data URIs |
| `'unsafe-inline'` | Inline scripts/styles (weakens CSP) |
| `'unsafe-eval'` | `eval()` and `new Function()` (weakens CSP) |
| `'nonce-{random}'` | Specific inline script/style by nonce |
| `'sha256-...'` | Specific inline script/style by hash |

## Static Site CSP (Hugo)

```
Content-Security-Policy: default-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; frame-ancestors 'none'
```

## Rollout Strategy

1. **Report-Only** (1-2 weeks): `Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-report`
2. **Lenient enforcing** (1 week): Allow `'unsafe-inline'`, monitor for breakage
3. **Remove unsafe-inline**: Switch to nonces/hashes
4. **Strict**: `default-src 'none'` with explicit allows

## Testing

- **Google CSP Evaluator:** https://csp-evaluator.withgoogle.com/
- **Browser console:** Shows violation messages with blocked URI and directive
- **SecurityHeaders.com:** https://securityheaders.com/

## Pitfalls to Avoid

- Using `'unsafe-inline'` for scripts (defeats CSP purpose)
- `script-src *` or `script-src https:` (too permissive)
- Deploying strict CSP without report-only testing first
- Missing `report-uri` during testing phase (no visibility into violations)
- Forgetting to update policy when adding third-party scripts

## Related

- [security-headers.md](./security-headers.md)
- [permissions-policy.md](./permissions-policy.md)
