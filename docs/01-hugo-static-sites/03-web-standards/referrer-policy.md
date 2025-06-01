# Referrer-Policy

HTTP header (and meta tag) controlling how much referrer information is sent when navigating away from your site.

## Why It Matters

- Prevents leaking sensitive URL paths and query parameters to third parties
- Protects user privacy (referrer can reveal browsing history)
- Default browser behavior (`no-referrer-when-downgrade`) sends full URLs to same-protocol destinations

## Recommended Value

```
Referrer-Policy: strict-origin-when-cross-origin
```

This sends the full URL to same-origin requests, only the origin (`https://example.com`) to cross-origin HTTPS, and nothing for HTTPS-to-HTTP downgrades.

## All Values

| Value | Same-Origin | Cross-Origin HTTPS | HTTPS to HTTP |
|-------|------------|-------------------|---------------|
| `no-referrer` | Nothing | Nothing | Nothing |
| `no-referrer-when-downgrade` | Full URL | Full URL | Nothing |
| `origin` | Origin only | Origin only | Origin only |
| `origin-when-cross-origin` | Full URL | Origin only | Origin only |
| `same-origin` | Full URL | Nothing | Nothing |
| `strict-origin` | Origin only | Origin only | Nothing |
| `strict-origin-when-cross-origin` | Full URL | Origin only | Nothing |
| `unsafe-url` | Full URL | Full URL | Full URL |

## NGINX

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

## HTML Meta Tag (Alternative)

```html
<meta name="referrer" content="strict-origin-when-cross-origin">
```

## Per-Link Override

```html
<a href="https://external.com" referrerpolicy="no-referrer">Link</a>
```

## Pitfalls to Avoid

- Using `unsafe-url` (leaks full URL including query strings to all destinations)
- `no-referrer` breaking analytics that depend on referrer data
- Forgetting that `<meta>` tag is overridden by the HTTP header if both are set

## Related

- [security-headers.md](./security-headers.md)
