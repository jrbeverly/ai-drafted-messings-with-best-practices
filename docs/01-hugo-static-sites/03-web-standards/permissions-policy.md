# Permissions Policy

HTTP header controlling which browser features and APIs (camera, microphone, geolocation) your site can use.

## Why It Matters

- Disables sensitive APIs you do not need, reducing attack surface
- Prevents third-party iframes from accessing device sensors
- Replaces the deprecated `Feature-Policy` header

## Recommended Baseline

```
Permissions-Policy: geolocation=(), microphone=(), camera=(), payment=(), usb=()
```

`()` = disabled for all origins. `(self)` = allowed for same origin only. `*` = allowed for all (not recommended).

## Common Features to Control

| Feature | Disable if you do not use it |
|---------|------------------------------|
| `geolocation` | Location tracking |
| `camera` | Camera access |
| `microphone` | Microphone access |
| `payment` | Payment Request API |
| `usb` | WebUSB |
| `accelerometer` | Device motion sensor |
| `gyroscope` | Device orientation sensor |
| `magnetometer` | Compass sensor |
| `autoplay` | Video/audio autoplay |
| `fullscreen` | Fullscreen API |
| `picture-in-picture` | PiP video |
| `sync-xhr` | Synchronous XMLHttpRequest |

## Per-Site-Type Policies

**Static site / blog:**
```
Permissions-Policy: geolocation=(), microphone=(), camera=(), payment=(), usb=(), fullscreen=(self), picture-in-picture=(self)
```

**E-commerce:**
```
Permissions-Policy: geolocation=(self), microphone=(), camera=(), payment=(self), usb=()
```

**Video chat app:**
```
Permissions-Policy: geolocation=(), microphone=(self), camera=(self), payment=(), usb=()
```

## CloudFront (Terraform)

```hcl
custom_headers_config {
  items {
    header   = "Permissions-Policy"
    value    = "geolocation=(), microphone=(), camera=(), payment=()"
    override = true
  }
}
```

## NGINX

```nginx
add_header Permissions-Policy "geolocation=(), microphone=(), camera=(), payment=(), usb=()" always;
```

## Pitfalls to Avoid

- Using deprecated `Feature-Policy` header (use `Permissions-Policy`)
- Defaulting to `*` (allows all origins -- too permissive)
- Blocking features you actually need (test fullscreen, autoplay, etc.)
- Forgetting that page-level policy also restricts iframes (more restrictive wins)

## Related

- [security-headers.md](./security-headers.md)
- [csp-content-security-policy.md](./csp-content-security-policy.md)
