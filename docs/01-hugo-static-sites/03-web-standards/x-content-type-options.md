# X-Content-Type-Options

HTTP header that prevents browsers from MIME-sniffing a response away from the declared Content-Type.

## Why It Matters

- Without it, browsers may "sniff" a file's type and execute it as a script
- An attacker could upload a `.txt` file containing JavaScript and have the browser execute it
- The only valid value is `nosniff` -- set it and forget it

## Header

```
X-Content-Type-Options: nosniff
```

There is only one valid value. No configuration choices needed.

## What It Prevents

- Browser interpreting a `text/plain` file as `text/html` and rendering it
- Executing a non-JavaScript file as a script (`<script src="/upload/malicious.txt">`)
- MIME confusion attacks where files are served with wrong Content-Type

## NGINX

```nginx
add_header X-Content-Type-Options "nosniff" always;
```

## CloudFront (Terraform)

```hcl
content_type_options {
  override = true
}
```

## Pitfalls to Avoid

- Omitting this header entirely (one of the easiest security wins)
- Relying on it alone without setting correct `Content-Type` on your responses
- Not using `always` in NGINX (header missing from error pages)

## Related

- [security-headers.md](./security-headers.md)
