# URL Rewriting

Rewrite URLs. Redirects. Canonical URLs. SEO-friendly URLs.

## Principle

Rewrite rules transform incoming URLs. Redirect old URLs to new ones. Improve SEO and user experience.

## Basic URL Rewriting

```csharp
// Install: Microsoft.AspNetCore.Rewrite (included in framework)

var builder = WebApplication.CreateBuilder(args);

var app = builder.Build();

// Configure rewrite rules
var rewriteOptions = new RewriteOptions()
    // Remove www prefix
    .AddRedirect("^www\\.(.+)", "https://$1", (int)HttpStatusCode.MovedPermanently)

    // Trailing slash redirect
    .AddRedirect("(.+)/$", "$1", (int)HttpStatusCode.MovedPermanently)

    // Custom rewrite (internal, no redirect)
    .AddRewrite(@"^product/(\d+)", "api/products/$1", skipRemainingRules: false);

app.UseRewriter(rewriteOptions);

app.MapGet("/", () => "Home");
app.MapGet("/api/products/{id}", (string id) => $"Product {id}");

app.Run();
```

## Redirect vs Rewrite

```csharp
var rewriteOptions = new RewriteOptions();

// Redirect: Client sees new URL (302/301 response)
rewriteOptions.AddRedirect(
    "old-page",
    "new-page",
    (int)HttpStatusCode.MovedPermanently);  // 301

// Rewrite: Server changes URL internally (client unaware)
rewriteOptions.AddRewrite(
    @"^product/(\d+)",
    "api/products/$1",
    skipRemainingRules: false);

app.UseRewriter(rewriteOptions);

// Client requests: /old-page
// → Browser redirected to: /new-page

// Client requests: /product/123
// → Server processes: /api/products/123
// → Client still sees: /product/123
```

## Common Rewrite Rules

```csharp
var rewriteOptions = new RewriteOptions();

// 1. Remove www prefix
rewriteOptions.AddRedirect(
    "^www\\.(.+)",
    "https://$1",
    (int)HttpStatusCode.MovedPermanently);

// 2. Force HTTPS
rewriteOptions.AddRedirectToHttps(301);

// 3. Remove trailing slashes
rewriteOptions.AddRedirect("(.+)/$", "$1");

// 4. Lowercase URLs
rewriteOptions.Add(new LowercaseUrlsRule());

// 5. Default document
rewriteOptions.AddRewrite("^$", "index.html", skipRemainingRules: true);

app.UseRewriter(rewriteOptions);
```

## Custom Rewrite Rule

```csharp
public class LowercaseUrlsRule : IRule
{
    public void ApplyRule(RewriteContext context)
    {
        var request = context.HttpContext.Request;
        var path = request.Path.ToString();

        if (path != path.ToLowerInvariant())
        {
            var response = context.HttpContext.Response;
            response.StatusCode = (int)HttpStatusCode.MovedPermanently;
            response.Headers.Location = $"{request.Scheme}://{request.Host}{path.ToLowerInvariant()}{request.QueryString}";

            context.Result = RuleResult.EndResponse;
        }
    }
}

// Usage
var rewriteOptions = new RewriteOptions();
rewriteOptions.Add(new LowercaseUrlsRule());
app.UseRewriter(rewriteOptions);
```

## SEO-Friendly URLs

```csharp
// Rewrite slug URLs to ID-based endpoints
var rewriteOptions = new RewriteOptions();

// /blog/my-first-post → /api/posts?slug=my-first-post
rewriteOptions.AddRewrite(
    @"^blog/([a-z0-9-]+)$",
    "api/posts?slug=$1",
    skipRemainingRules: true);

// /category/technology → /api/posts?category=technology
rewriteOptions.AddRewrite(
    @"^category/([a-z0-9-]+)$",
    "api/posts?category=$1",
    skipRemainingRules: true);

app.UseRewriter(rewriteOptions);

app.MapGet("/api/posts", (string? slug, string? category) =>
{
    if (slug is not null)
    {
        return Results.Ok(new { Slug = slug });
    }

    if (category is not null)
    {
        return Results.Ok(new { Category = category });
    }

    return Results.BadRequest();
});
```

## Conditional Rewrites

```csharp
public class MobileRewriteRule : IRule
{
    public void ApplyRule(RewriteContext context)
    {
        var request = context.HttpContext.Request;
        var userAgent = request.Headers.UserAgent.ToString().ToLowerInvariant();

        // Detect mobile
        if (userAgent.Contains("mobile") || userAgent.Contains("android") || userAgent.Contains("iphone"))
        {
            // Rewrite to mobile site
            var path = request.Path.ToString();

            if (!path.StartsWith("/m/"))
            {
                request.Path = $"/m{path}";
            }
        }
    }
}

// Usage
var rewriteOptions = new RewriteOptions();
rewriteOptions.Add(new MobileRewriteRule());
app.UseRewriter(rewriteOptions);

// Desktop: /products → /products
// Mobile:  /products → /m/products
```

## Redirect Old URLs

```csharp
// Migrate old URL structure to new
var rewriteOptions = new RewriteOptions();

// Old product URLs
rewriteOptions.AddRedirect(
    @"^products\.php\?id=(\d+)",
    "products/$1",
    (int)HttpStatusCode.MovedPermanently);

// Old blog URLs
rewriteOptions.AddRedirect(
    @"^blog\.php\?post=(\d+)",
    "blog/$1",
    (int)HttpStatusCode.MovedPermanently);

// Old category URLs
rewriteOptions.AddRedirect(
    @"^category\.php\?name=([^&]+)",
    "categories/$1",
    (int)HttpStatusCode.MovedPermanently);

app.UseRewriter(rewriteOptions);

// /products.php?id=123 → 301 redirect to /products/123
// /blog.php?post=456 → 301 redirect to /blog/456
```

## IIS URL Rewrite Rules (Import)

```xml
<!-- web.config -->
<rewrite>
  <rules>
    <rule name="Force HTTPS" stopProcessing="true">
      <match url="(.*)" />
      <conditions>
        <add input="{HTTPS}" pattern="off" />
      </conditions>
      <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" redirectType="Permanent" />
    </rule>

    <rule name="Remove www" stopProcessing="true">
      <match url="(.*)" />
      <conditions>
        <add input="{HTTP_HOST}" pattern="^www\.(.+)$" />
      </conditions>
      <action type="Redirect" url="https://{C:1}/{R:1}" redirectType="Permanent" />
    </rule>
  </rules>
</rewrite>
```

```csharp
// Import IIS rewrite rules
var rewriteOptions = new RewriteOptions();

using var iisUrlRewriteStreamReader = File.OpenText("IISUrlRewrite.xml");
rewriteOptions.AddIISUrlRewrite(iisUrlRewriteStreamReader);

app.UseRewriter(rewriteOptions);
```

## Apache mod_rewrite Rules (Import)

```apache
# .htaccess
RewriteEngine On

# Force HTTPS
RewriteCond %{HTTPS} off
RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]

# Remove www
RewriteCond %{HTTP_HOST} ^www\.(.+)$ [NC]
RewriteRule ^ https://%1%{REQUEST_URI} [L,R=301]
```

```csharp
// Import Apache mod_rewrite rules
var rewriteOptions = new RewriteOptions();

using var apacheModRewriteStreamReader = File.OpenText(".htaccess");
rewriteOptions.AddApacheModRewrite(apacheModRewriteStreamReader);

app.UseRewriter(rewriteOptions);
```

## Canonical URL Enforcement

```csharp
public class CanonicalUrlRule : IRule
{
    private readonly string _canonicalHost;

    public CanonicalUrlRule(string canonicalHost)
    {
        _canonicalHost = canonicalHost;
    }

    public void ApplyRule(RewriteContext context)
    {
        var request = context.HttpContext.Request;
        var host = request.Host.Host;

        // Enforce canonical host
        if (!host.Equals(_canonicalHost, StringComparison.OrdinalIgnoreCase))
        {
            var response = context.HttpContext.Response;
            response.StatusCode = (int)HttpStatusCode.MovedPermanently;
            response.Headers.Location = $"{request.Scheme}://{_canonicalHost}{request.Path}{request.QueryString}";

            context.Result = RuleResult.EndResponse;
        }

        // Enforce lowercase URLs
        var path = request.Path.ToString();
        if (path != path.ToLowerInvariant())
        {
            var response = context.HttpContext.Response;
            response.StatusCode = (int)HttpStatusCode.MovedPermanently;
            response.Headers.Location = $"{request.Scheme}://{_canonicalHost}{path.ToLowerInvariant()}{request.QueryString}";

            context.Result = RuleResult.EndResponse;
        }
    }
}

// Usage
var rewriteOptions = new RewriteOptions();
rewriteOptions.Add(new CanonicalUrlRule("example.com"));
app.UseRewriter(rewriteOptions);

// www.example.com → 301 to example.com
// EXAMPLE.COM → 301 to example.com
// /Products → 301 to /products
```

## Performance Considerations

```csharp
// Cache compiled regex patterns
public class CachedRegexRule : IRule
{
    private readonly Regex _regex;
    private readonly string _replacement;
    private readonly int _statusCode;

    public CachedRegexRule(string pattern, string replacement, int statusCode)
    {
        // Compiled regex (faster)
        _regex = new Regex(pattern, RegexOptions.Compiled | RegexOptions.IgnoreCase);
        _replacement = replacement;
        _statusCode = statusCode;
    }

    public void ApplyRule(RewriteContext context)
    {
        var request = context.HttpContext.Request;
        var path = request.Path.ToString();

        var match = _regex.Match(path);

        if (match.Success)
        {
            var newPath = _regex.Replace(path, _replacement);
            var response = context.HttpContext.Response;

            response.StatusCode = _statusCode;
            response.Headers.Location = $"{request.Scheme}://{request.Host}{newPath}{request.QueryString}";

            context.Result = RuleResult.EndResponse;
        }
    }
}
```

## Testing Rewrite Rules

```csharp
public class RewriteRuleTests
{
    [Theory]
    [InlineData("/old-page", "/new-page")]
    [InlineData("/product/123", "/api/products/123")]
    public async Task RewriteRule_RedirectsCorrectly(string input, string expected)
    {
        // Arrange
        var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false
        });

        // Act
        var response = await client.GetAsync(input);

        // Assert
        Assert.Equal(HttpStatusCode.MovedPermanently, response.StatusCode);
        Assert.Contains(expected, response.Headers.Location?.ToString());
    }
}
```

## Guidelines

**SEO Best Practices:**
- Use 301 (permanent) redirects for moved content
- Lowercase URLs for consistency
- Remove www or enforce www (choose one)
- No trailing slashes (or always trailing slashes)
- Force HTTPS

**Performance:**
- Use compiled regex patterns
- Order rules from specific to general
- Use `skipRemainingRules` when appropriate
- Cache redirect mappings if from database

**Security:**
- Validate redirect targets (no open redirects)
- Don't expose internal paths in rewrites
- Sanitize user input in custom rules

**Maintenance:**
- Document rewrite rules
- Test all rules
- Monitor 404 errors (missed rewrites)
- Keep rules simple and readable

## Benefits

SEO-friendly. Clean, readable URLs.

Backward-compatible. Redirect old URLs.

Flexible. Regex-based pattern matching.

Performance. Server-side (no client redirects).

## Related

- [routing-basics.md](../04-aspnet-web-api/routing-basics.md) - Routing
- [middleware-basics.md](../04-aspnet-web-api/middleware-basics.md) - Middleware order
- [seo-best-practices.md](../../02-frontend/seo-best-practices.md) - SEO optimization
- [security-headers.md](../../09-security/security-headers.md) - HTTPS enforcement
