# ads.txt

IAB standard declaring which companies are authorized to sell your ad inventory. Combats ad fraud and domain spoofing.

## Why It Matters

- Prevents unauthorized reselling of your ad inventory
- Required by Google AdSense/Ad Manager for programmatic ads
- Ad buyers verify legitimacy by checking this file

## Do You Need It?

**Yes** if you run Google AdSense, Ad Manager, or any programmatic ads. **No** if you have no ads.

## Format

```text
# domain, publisher_id, relationship, cert_authority_id
google.com, pub-1234567890123456, DIRECT, f08c47fec0942fa0
appnexus.com, 12345, RESELLER, f5ab79cb980f11d1
```

| Field | Description |
|-------|-------------|
| domain | Ad system domain (e.g., `google.com`) |
| publisher_id | Your account ID with that system |
| relationship | `DIRECT` (your account) or `RESELLER` (managed by third party) |
| cert_authority_id | Optional TAG ID from tagtoday.net |

## Special Records

```text
contact=ads@example.com
subdomain=blog.example.com
```

## Hugo Setup

Place at `static/ads.txt`. Hugo copies it to site root automatically.

## Serving Requirements

- **Location:** `/ads.txt` (root of domain, not a subdirectory)
- **MIME type:** `text/plain; charset=utf-8`
- **Response:** 200 OK

## Subdomain Handling

Either redirect `blog.example.com/ads.txt` to `example.com/ads.txt`, or add `subdomain=example.com` in the subdomain's file.

## Pitfalls to Avoid

- File not at domain root (`/static/ads.txt` instead of `/ads.txt`)
- Wrong MIME type (serving as `text/html`)
- Including `https://` in the domain field (use bare domain)
- Lowercase relationship type (`direct` instead of `DIRECT`)
- Missing publisher ID field

## Validation

- IAB validator: https://adstxt.guru/
- Google validator: https://adstxt.admanager.google.com/

## Related

- [app-ads-txt.md](./app-ads-txt.md)
- [sellers-json.md](./sellers-json.md)
