# Cross-Origin Headers

HTTP headers controlling how your site interacts with cross-origin resources: embedding, opener relationships, and resource sharing.

## Why It Matters

- Isolates your site from cross-origin attacks (Spectre, side-channel)
- Controls whether other sites can embed your resources
- Required for advanced APIs like `SharedArrayBuffer`

## Key Headers

### Cross-Origin-Embedder-Policy (COEP)

Controls whether your page can load cross-origin resources without explicit permission.

```
Cross-Origin-Embedder-Policy: require-corp
```

| Value | Meaning |
|-------|---------|
| `unsafe-none` | Default, no restrictions |
| `require-corp` | All cross-origin resources must opt in via CORP or CORS |
| `credentialless` | Cross-origin requests sent without credentials |

### Cross-Origin-Opener-Policy (COOP)

Controls whether other windows/tabs can get a reference to your window.

```
Cross-Origin-Opener-Policy: same-origin
```

| Value | Meaning |
|-------|---------|
| `unsafe-none` | Default, no restrictions |
| `same-origin` | Isolate browsing context from cross-origin openers |
| `same-origin-allow-popups` | Isolate, but allow popups to retain reference |

### Cross-Origin-Resource-Policy (CORP)

Controls who can load your resources (images, scripts, etc.).

```
Cross-Origin-Resource-Policy: same-origin
```

| Value | Meaning |
|-------|---------|
| `same-origin` | Only same-origin pages can load this resource |
| `same-site` | Same-site pages can load (includes subdomains) |
| `cross-origin` | Any origin can load this resource |

## Enabling Cross-Origin Isolation

To access `SharedArrayBuffer` and high-resolution timers, set both:

```
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Opener-Policy: same-origin
```

Verify in JavaScript: `self.crossOriginIsolated === true`

## NGINX

```nginx
add_header Cross-Origin-Opener-Policy "same-origin" always;
add_header Cross-Origin-Embedder-Policy "require-corp" always;
add_header Cross-Origin-Resource-Policy "same-origin" always;
```

## Pitfalls to Avoid

- Setting `require-corp` breaks loading of third-party images/scripts that lack CORP/CORS headers
- Forgetting that `same-origin` COOP breaks OAuth popups and payment flows
- Applying CORP `same-origin` on a public CDN (blocks all cross-origin consumers)
- Enabling cross-origin isolation without testing all embedded resources first

## Related

- [security-headers.md](./security-headers.md)
- [csp-content-security-policy.md](./csp-content-security-policy.md)
