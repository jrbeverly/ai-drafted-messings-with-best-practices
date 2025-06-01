# humans.txt

Community convention (not a formal standard) for crediting the people and tools behind a website.

## Why It Matters

- Easter egg for curious developers and recruiters
- Documents your tech stack publicly
- Humanizes your website (robots.txt for bots, humans.txt for people)

## Basic Example

```text
/* TEAM */

Name: Jane Doe
Role: Developer
Contact: jane@example.com
From: San Francisco, CA

/* THANKS */

Special thanks to the open-source community.

/* SITE */

Last update: 2026-02-15
Standards: HTML5, CSS3
Components: Hugo, Vue.js
```

## Common Sections

| Section | Content |
|---------|---------|
| `/* TEAM */` | Names, roles, contact info |
| `/* THANKS */` | Acknowledgments and credits |
| `/* SITE */` | Tech stack, last update, standards |
| `/* TECHNOLOGY COLOPHON */` | Detailed tech breakdown |

## HTML Discovery Tag

```html
<link rel="author" href="/humans.txt" type="text/plain">
```

## Hugo Setup

Place at `static/humans.txt`. Hugo copies it to site root.

## Serving Requirements

- **Location:** `/humans.txt` (root)
- **MIME type:** `text/plain; charset=utf-8`
- **Cache:** 1 day

## Pitfalls to Avoid

- Including sensitive information (passwords, API keys, private phone numbers)
- Wrong location (must be at domain root)
- Stale "Last update" dates (automate via CI/CD if possible)
- Overthinking format (there is no strict spec -- keep it readable)

## Related

- [security-txt.md](./security-txt.md)
- [well-known-directory.md](./well-known-directory.md)
