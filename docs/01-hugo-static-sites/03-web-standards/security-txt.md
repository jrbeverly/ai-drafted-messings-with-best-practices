# security.txt

RFC 9116 standard file telling security researchers how to report vulnerabilities in your site.

## Why It Matters

- Security researchers **expect** this file at `/.well-known/security.txt`
- Enables coordinated vulnerability disclosure
- Required fields: `Contact` and `Expires` (must be < 1 year out)

## Minimal Example

```text
Contact: mailto:security@example.com
Expires: 2027-12-31T23:59:59.000Z
Preferred-Languages: en
Canonical: https://example.com/.well-known/security.txt
```

## All Fields

| Field | Required | Purpose |
|-------|----------|---------|
| `Contact` | Yes | Email (`mailto:`), URL, or phone |
| `Expires` | Yes | RFC 3339 date, < 1 year from now |
| `Encryption` | Recommended | PGP key URL or fingerprint |
| `Canonical` | Recommended | Authoritative URL of this file |
| `Preferred-Languages` | Optional | ISO 639-1 codes (e.g., `en, es`) |
| `Policy` | Optional | Link to full disclosure policy |
| `Acknowledgments` | Optional | Hall of fame URL |
| `Hiring` | Optional | Security job postings URL |

## Hugo Setup

Place at `static/.well-known/security.txt`. Hugo copies it as-is to the build output.

## Serving Requirements

- **MIME type:** `text/plain; charset=utf-8`
- **Protocol:** Must be accessible via HTTPS
- **Cache:** 1-24 hours recommended

## PGP Signing (Optional)

```bash
gpg --clearsign --output security.txt.sig security.txt
```

## Validation

- Online: https://securitytxt.org/
- CLI: `curl https://example.com/.well-known/security.txt`

## Pitfalls to Avoid

- Missing `Expires` field (required by spec)
- `Expires` date more than 1 year in the future (invalid)
- Serving as `text/html` instead of `text/plain`
- Forgetting to update before expiration
- HTTP-only access (must be HTTPS)

## Related

- [well-known-directory.md](./well-known-directory.md)
