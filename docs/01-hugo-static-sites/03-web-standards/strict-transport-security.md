# Strict-Transport-Security (HSTS)

HTTP header that forces browsers to use HTTPS for all future connections to your domain, preventing protocol downgrade attacks.

## Why It Matters

- Prevents SSL stripping attacks (MITM intercepting HTTP and blocking upgrade to HTTPS)
- After first visit, browser refuses to connect over HTTP for the specified duration
- HSTS preload list eliminates even the first-visit vulnerability

## Recommended Value

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

## Parameters

| Parameter | Meaning |
|-----------|---------|
| `max-age=31536000` | Remember for 1 year (in seconds) |
| `includeSubDomains` | Apply to all subdomains |
| `preload` | Eligible for browser preload list submission |

## NGINX

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

## CloudFront (Terraform)

```hcl
strict_transport_security {
  override                   = true
  access_control_max_age_sec = 31536000
  include_subdomains         = true
  preload                    = true
}
```

## HSTS Preload List

Browsers ship with a preload list so HTTPS is enforced even on the very first visit:

1. Ensure valid HTTPS cert on root domain and all subdomains
2. Redirect all HTTP to HTTPS
3. Serve HSTS header with `max-age >= 31536000`, `includeSubDomains`, `preload`
4. Submit at https://hstspreload.org/

**Warning:** Removal from the preload list takes months. All subdomains must support HTTPS before enabling `includeSubDomains`.

## Gradual Rollout

Start with a short `max-age` and increase:

1. `max-age=300` (5 minutes) -- test
2. `max-age=86400` (1 day) -- monitor
3. `max-age=2592000` (30 days) -- verify
4. `max-age=31536000` (1 year) -- production

## Pitfalls to Avoid

- Enabling HSTS before HTTPS is fully working (locks users out of HTTP)
- `includeSubDomains` when some subdomains do not support HTTPS
- Submitting to preload list before you are committed (hard to undo)
- Serving HSTS header over HTTP (browsers must ignore it; only valid over HTTPS)
- Starting with `max-age=31536000` without testing first

## Related

- [security-headers.md](./security-headers.md)
