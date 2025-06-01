# X-Frame-Options

HTTP header that controls whether your site can be embedded in an `<iframe>`, preventing clickjacking attacks.

## Why It Matters

- Prevents clickjacking: an attacker overlays your site in a hidden iframe to trick users into clicking
- Simple to set -- one header, one value for most sites
- Being superseded by CSP `frame-ancestors` but still widely used for backward compatibility

## Recommended Value

```
X-Frame-Options: DENY
```

## All Values

| Value | Meaning |
|-------|---------|
| `DENY` | Never allow framing (recommended for most sites) |
| `SAMEORIGIN` | Allow framing only by pages on the same origin |
| `ALLOW-FROM https://example.com` | Allow framing by specific origin (deprecated, not supported in modern browsers) |

## Modern Replacement: CSP frame-ancestors

```
Content-Security-Policy: frame-ancestors 'none'
```

CSP `frame-ancestors` is more flexible and supports multiple origins. Use both headers for maximum compatibility:

```
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
```

## NGINX

```nginx
add_header X-Frame-Options "DENY" always;
```

## CloudFront (Terraform)

```hcl
frame_options {
  override     = true
  frame_option = "DENY"
}
```

## When to Use SAMEORIGIN

Use `SAMEORIGIN` if your site legitimately iframes itself (e.g., preview panes, embedded components):

```
X-Frame-Options: SAMEORIGIN
Content-Security-Policy: frame-ancestors 'self'
```

## Pitfalls to Avoid

- Omitting the header entirely (enables clickjacking)
- Using `ALLOW-FROM` (deprecated, ignored by Chrome/Firefox)
- Setting `DENY` when your site actually needs to be iframed by same-origin pages
- Not also setting `frame-ancestors` in CSP for modern browser coverage

## Related

- [security-headers.md](./security-headers.md)
- [csp-content-security-policy.md](./csp-content-security-policy.md)
