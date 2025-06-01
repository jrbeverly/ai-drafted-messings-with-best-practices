# Change Password URL

A well-known URL (`/.well-known/change-password`) that redirects to your site's password change form, enabling password managers to link users directly there.

## Why It Matters

- Password managers (1Password, Chrome, Safari) use this URL to offer "Change Password" buttons
- Improves security UX by removing friction from password rotation
- Simple redirect -- no API or complex setup needed

## How It Works

Browser or password manager requests `https://example.com/.well-known/change-password` and expects a redirect (302/303) to the actual password change page.

## Implementation

### Static Redirect (Hugo + CloudFront/NGINX)

**NGINX:**
```nginx
location = /.well-known/change-password {
    return 302 https://example.com/account/password;
}
```

**CloudFront Function or S3 redirect rule** can also handle this.

### Hugo Static File (Fallback)

If you cannot configure redirects, place an HTML file with a meta refresh at `static/.well-known/change-password`:

```html
<!DOCTYPE html>
<meta http-equiv="refresh" content="0;url=https://example.com/account/password">
```

## Specification

- W3C: https://w3c.github.io/webappsec-change-password-url/
- Response: 302 or 303 redirect to actual password form
- Must be on the same origin

## Pitfalls to Avoid

- Returning 200 instead of a redirect (password managers expect 3xx)
- Redirecting to a different domain (must stay same-origin)
- Adding a file extension (`change-password.html` -- must be extensionless)

## Related

- [well-known-directory.md](./well-known-directory.md)
- [security-txt.md](./security-txt.md)
